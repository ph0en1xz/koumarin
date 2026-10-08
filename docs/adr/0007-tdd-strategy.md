# ADR 0007 — TDD strategy and test stack

- **Status:** Accepted
- **Date:** 2026-10-08
- **Deciders:** Owner (Mario), Claude
- **Scope:** How Koumarin is tested, and the stack used. Records the TDD requirement stated on
  2026-10-08. Depends on ADR 0001 (layering) and ADR 0004 (what must be proven).

## Context

TDD is a requirement, not a preference. Two things decide whether it is actually practicable here:

- **Speed.** A red-green-refactor loop that waits on a Docker container is not a loop. ADR 0001's
  layering is what makes speed achievable: the Domain layer has no I/O, so its tests need no
  database, no broker, and no host.
- **Truthfulness.** The guarantees in ADR 0004 are *database* guarantees — a unique constraint
  under concurrency, `SELECT … FOR UPDATE`, and the atomicity of three inserts. A test substrate
  that does not enforce those cannot prove the thing that matters.

These pull in opposite directions, so the strategy has to be layered rather than uniform.

## Decision

### Stack

- **Runner:** xUnit, via `dotnet test`.
- **Assertions:** FluentAssertions. **Mocking:** NSubstitute (Moq acceptable; pick one per repo).
- **Integration DB:** **Testcontainers (`Testcontainers.PostgreSql`)** against real Postgres,
  running the real ADR 0003 migrations. **Respawn** to reset between tests.
- **HTTP/API:** `WebApplicationFactory<Program>` — exercises the real middleware pipeline,
  including HMAC verification and model binding.
- **Provider fakes:** a wire-level HTTP stub for CasinoApiPro. Tests never hit the live sandbox;
  sandbox calls are for manual exploration only.
- **Time:** `TimeProvider` injected everywhere; `FakeTimeProvider` in tests. `DateTime.UtcNow` is
  never called directly — HMAC freshness windows and reminder logic are time-dependent and must be
  deterministic.
- **Architecture tests:** NetArchTest (or equivalent) asserting ADR 0001's dependency rule and ADR
  0006's constraint that the Orchestrator declares no aggregates.

### The EF Core InMemory provider is banned

Not discouraged — **banned**, on every test, including Reporting's read models.

It enforces neither unique indexes, nor foreign keys, nor transactions. It would therefore pass
the exact tests that protect this system's money: a duplicate `transaction_id` would insert
cleanly, and a rolled-back transaction would leave partial state. It produces green tests for a
system that is broken in production, which is worse than having no test.

SQLite in-memory is rejected for related reasons: no `FOR UPDATE`, different `numeric` semantics,
and no parity on `ON CONFLICT` behaviour.

### Test layers

| Layer | Substrate | Speed | What it proves |
|---|---|---|---|
| **Domain** | None — pure objects | milliseconds | Business rules and invariants. The TDD loop lives here. |
| **Application** | Fake repositories | milliseconds | Use-case orchestration, error-code mapping |
| **Infrastructure** | Testcontainers Postgres + real migrations | seconds | SQL, locking, constraints, atomicity |
| **Api** | `WebApplicationFactory` | seconds | HMAC, routing, serialisation, status codes |
| **Architecture** | Reflection | milliseconds | Layering rules cannot silently erode |

### Required money-path test recipes

These are not optional coverage; they are the specification of ADR 0004 and must exist as tests:

1. **Concurrent duplicate `transactionId`** — two parallel debits with the same id: exactly one
   ledger row, one balance movement, and both callers receive the same balance.
2. **Replay after intervening movement** — debit A, then debit B, then retry A: the retry returns
   A's **stored `balance_after`**, not a recomputed balance. This is the test that protects the
   subtlety most likely to be lost in a refactor.
3. **Timeout then retry** — simulate a lost response; assert the retry is a replay, not a second
   movement.
4. **Insufficient funds** — rejected with `insufficient_funds`, no ledger row, balance unchanged.
5. **Blocked player via the replica** — `player_blocked` returned from the local replica with no
   call to the Player service.
6. **Stale replica** — an out-of-order `player.*` event is ignored rather than applied.
7. **Refund of an unknown reference** → `transaction_not_found`; **double refund** →
   `already_reversed`.
8. **Byte-exact HMAC** — a signature computed over the raw body verifies; a re-serialised body
   fails. This guards the trap where ASP.NET Core re-serialisation silently breaks every callback.
9. **Ledger/balance identity** — `SUM` over a player's ledger equals `accounts.balance` after a
   randomised sequence of operations.
10. **Outbox atomicity** — a rolled-back transaction leaves no outbox row; a committed one leaves
    exactly one.

## Consequences

- **Positive:** the domain loop is genuinely fast, so TDD is practised rather than claimed.
- **Positive:** the guarantees that protect money are proven against real Postgres, including
  under concurrency. Recipe 1 is only meaningful on a substrate with a real unique index.
- **CI requires a Docker daemon.** Integration tests will not run without it. This is satisfied
  locally by the WSL Docker daemon (28.5.1) and on GitHub Actions `ubuntu-latest`, which ships
  Docker. Noted because it is a hard CI dependency, not a nicety.
- **Integration tests are slower**, seconds rather than milliseconds. Mitigated by keeping them to
  the Infrastructure and Api layers, sharing one container per test collection, and using Respawn
  rather than recreating the container per test.
- **Reporting's read-model tests also need real Postgres**, as a direct consequence of the InMemory
  ban. This is deliberate: it is also what detects EF Core read contexts drifting from the
  hand-written schema (ADR 0002, ADR 0003).
- **Writing tests first will surface design problems in the domain model early**, which is most of
  the value — particularly around where the idempotency rule can and cannot be enforced (ADR 0004).

## Alternatives considered

- **EF Core InMemory provider.** Fast, zero dependencies, no Docker. Rejected and banned: it does
  not enforce unique indexes or transactions, so it would pass the system's most important tests
  while production fails them. The canonical false-confidence trap for exactly this kind of system.
- **SQLite in-memory.** A real SQL engine and much faster than a container. Rejected: no
  `SELECT … FOR UPDATE`, different numeric/decimal semantics, and insufficient `ON CONFLICT`
  parity — so the locking and constraint behaviour under test would not be the behaviour shipped.
- **A shared long-lived dev/test Postgres database.** Fast, no container startup. Rejected: tests
  pollute each other, parallel runs conflict, and CI depends on a stateful external resource.
- **Mocking the repository interfaces for all tests, no database at all.** Fast and comprehensive
  against the *interfaces*. Rejected: it proves the code calls the repository correctly while
  proving nothing about the SQL, which is where ADR 0004's guarantees actually live.
- **E2E tests against the live CasinoApiPro sandbox.** Valuable once, for confidence that the
  contract is understood. Rejected as an automated gate: non-deterministic, rate-limited, and
  dependent on a third party being up. The sandbox is for manual exploration; wire-level stubs are
  for CI.
