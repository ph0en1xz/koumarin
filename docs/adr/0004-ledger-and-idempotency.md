# ADR 0004 — Ledger and idempotency model

- **Status:** Accepted
- **Date:** 2026-10-08
- **Deciders:** Owner (Mario), Claude
- **Scope:** How Koumarin records money movement and guarantees that a provider callback is applied
  at most once. The most important decision in the system — together with ADR 0005 it is what
  protects the money.

## Context

Koumarin is a **seamless wallet**: CasinoApiPro calls our API for every bet and win, and waits on
our response body for the resulting balance. Their contract imposes three requirements directly:

1. **Idempotency on `transactionId`.** If we have seen the id, we must return *the same balance we
   returned the first time* and must not move money again.
2. **Atomicity.** The movement and the record of its id must be applied in one database
   transaction.
3. **Speed.** 6-second timeout, two retries with exponential backoff.

Requirement 3 is what makes requirement 1 load-bearing rather than defensive. **A timeout is
ambiguous**: the provider cannot distinguish "the debit never happened" from "the debit happened
and the response was lost." They will retry, and that retry may be a duplicate of work already
committed. Idempotency is therefore not an edge case — it is the normal operation of a lossy
network.

**Note added 2026-10-08:** per ADR 0006 §Amendment, the callback does not arrive at Wallet
directly. It is received and HMAC-verified by **Integration**, translated, and handed to Wallet via
the **Orchestrator**, with `transaction_id` carried through unchanged. This widens the scope of
this ADR rather than changing it: there are now three hops where a timeout is ambiguous instead of
one, and the same unique-index guarantee makes a retry at *any* of them safe. Everything below —
the write sequence, the replay path, and where the invariant lives — is internal to Wallet and
unaffected.

A further subtlety: by the time a retry arrives, *other* movements for the same player may have
committed. Recomputing the balance at replay time would return a different number than the first
response, violating requirement 1 even though no money moved twice.

## Decision

### Shape

- **`ledger_entries` is append-only.** One row per accepted movement. Never updated, never deleted.
  A reversal is a *new* row referencing the original, not a mutation of it.
- **`accounts` holds a cached balance** plus a `version` column. The ledger is the truth; the
  account row is a materialised convenience so the hot path does not aggregate history.
- **`UNIQUE (transaction_id)` on `ledger_entries`** is the guarantee. Everything else is
  bookkeeping around it.
- **`balance_after` is stored on every ledger row** — the exact balance returned to the provider
  for that transaction.
- Amounts are `numeric(19,4)`, **always positive**; direction is carried by `entry_type`
  (`debit` | `credit` | `refund` | `rollback`).
- `transaction_id` is **provider-generated and opaque**. Never parsed, never assumed to encode a
  round, a game, or a time.

### The write sequence (one transaction)

```
BEGIN
  SELECT … FOR UPDATE on accounts WHERE player_id = ?      -- serialises this player only
  check player_status_replica                              -- → player_blocked / player_not_found
  domain: Account.Debit(Money)                             -- → insufficient_funds
  INSERT INTO ledger_entries (…, transaction_id, balance_after, …)
  UPDATE accounts SET balance = …, version = version + 1
  INSERT INTO outbox (…)                                   -- ADR 0005
COMMIT
```

### The replay path

A unique-constraint violation on `transaction_id` (Postgres SQLSTATE **23505**) is **expected
control flow, not an error**. On catching it: roll back, select the existing ledger row by
`transaction_id`, and return **its stored `balance_after`** with `duplicate: true`. No money moves.

### Where the invariant lives

This deliberately spans two layers, and that is the decision:

| Layer | Role |
|---|---|
| **Domain** | *States* the rule: a `transactionId` is applied at most once; a balance may not go negative; a reversal must reference a real unreversed transaction. |
| **Infrastructure** | *Enforces* at-most-once atomically, via the unique index. This is the only race-free enforcement point. |
| **Application** | *Translates* a unique violation into the idempotent replay response. |

An aggregate cannot enforce uniqueness under concurrency — two requests can both load state, both
see no prior transaction, and both proceed. Only the database can arbitrate. **Do not "fix" this by
moving enforcement into the aggregate**, and do not replace it with a pre-check.

### Reversals

`refund` and `rollback` carry `reference_transaction_id`. Within the same transaction: verify the
referenced entry exists (else `transaction_not_found`) and has not already been reversed (else
`already_reversed`). Both checks are domain rules; the "already reversed" condition is additionally
protected by a partial unique index on `reference_transaction_id`, for the same reason as above.

## Consequences

- **Positive:** correctness under the provider's actual failure mode. A retried debit after a lost
  response returns the original balance and moves nothing.
- **Positive:** `FOR UPDATE` serialises **per player**, so concurrent bets by different players
  never contend. Contention is bounded by one player's own bet rate.
- **Positive:** the cached balance is always reconcilable — `SUM` over a player's ledger must equal
  `accounts.balance`. That identity is the reconciliation check (and a test assertion), and it
  exists because the ledger is append-only.
- **`balance_after` is non-negotiable.** Without it, a replay arriving after intervening movements
  would return a freshly computed balance and silently break the contract. This is the subtlety
  most easily lost in a later refactor.
- **A unique violation is normal.** Logging it as an error will produce noise and train people to
  ignore it; it should be counted as a metric (`wallet.duplicate_replay`) instead.
- **Cost accepted:** a write-heavy append-only table grows without bound. Fine at demo volume;
  partitioning by month is the documented later move, and the append-only shape is what makes
  partitioning easy.
- **Lock discipline:** every path that touches a player's money must take the account lock
  **first**, in the same order, or two concurrent operations can deadlock. Single lock, fixed
  order, no exceptions.

## Alternatives considered

- **Check-then-insert without a unique constraint.** Select by `transaction_id`, insert if absent.
  Rejected: a textbook race. Two concurrent retries both see nothing and both insert. This is the
  bug the whole ADR exists to prevent.
- **`SERIALIZABLE` isolation instead of row locks.** Correct, and pushes the problem onto
  serialisation failures that must then be retried at the application layer. Rejected: `FOR UPDATE`
  plus a unique index achieves the same guarantee with predictable behaviour and no retry loop.
- **Deriving the balance from `SUM(ledger_entries)` on every read.** Perfectly consistent and
  needs no cached balance or `version`. Rejected for the hot path — cost grows with a player's
  history inside a 6-second budget. It is retained as the **reconciliation query**, which is the
  right place for it.
- **Event sourcing the wallet.** A natural fit conceptually: the ledger already *is* an event log.
  Rejected as unnecessary — the provider wants a balance synchronously, so a projection would have
  to be strongly consistent with the log anyway, which is what the cached balance already does with
  far less machinery.
- **Idempotency keys in Redis instead of Postgres.** Faster, and loses the atomicity requirement:
  the key and the money would live in two systems with no shared transaction. Rejected outright.
  Redis is used only for the HMAC replay window, where losing an entry is survivable.
