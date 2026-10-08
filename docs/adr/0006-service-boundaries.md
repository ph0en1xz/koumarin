# ADR 0006 — Service boundaries: five services, and what each may not do

- **Status:** Accepted
- **Date:** 2026-10-08
- **Deciders:** Owner (Mario), Claude
- **Scope:** The decomposition of Koumarin into services, the data each owns, and the constraints
  that keep the decomposition safe. Supersedes an earlier three-service proposal from the same day.

## Amendment — 2026-10-08: the provider boundary moved out of Wallet

**Constraints 1 and 2 as originally written below are superseded.** Read this first.

The original text had CasinoApiPro calling Wallet directly, while simultaneously asserting that
Integration is the only service that knows CasinoApiPro exists. Those two statements contradict
each other: Wallet would have held provider HMAC, provider field names, and the provider's error
vocabulary. Mario caught the inconsistency; the boundary rule wins.

**The settlement path is now:**

```
CasinoApiPro → Integration → Orchestrator → Wallet
```

**Revised rules:**

1. **Integration is the only service that touches CasinoApiPro, in both directions.** All inbound
   callbacks and all outbound calls terminate there. It owns HMAC verification on the raw bytes,
   payload translation, error-dialect translation, DTOs, and provider resilience policy (ADR 0008).
2. **Wallet has no public ingress.** It is internal-only and accepts traffic from the Orchestrator
   alone. The internet cannot address the ledger.
3. **The Orchestrator is the only coordinator**, for provider-driven settlement and player-driven
   flows alike. It still holds **zero business invariants**: routing a `balance` callback to a read
   and a `debit` callback to a write is message routing — a *sequence*. Whether the player can
   afford it, is blocked, or is a duplicate stays in Wallet's domain.
4. **Idempotency on `transaction_id` now also protects the internal hops.** A timeout between our
   own services is as ambiguous as one from the provider, and the correct response to each is a
   retry with the same id (ADR 0004).

**What this costs, accepted knowingly:** three services must be up to settle a bet rather than one.
Latency is not the concern — two in-cluster hops are single-digit milliseconds against a 6,000 ms
budget, and the original objection to this change overstated latency as the problem. The real cost
is that **availability compounds**, so Integration and the Orchestrator become as
availability-critical as Wallet: multiple replicas, pod disruption budgets, and no heavy
dependencies on the settlement path.

**What it buys, beyond consistency:** HMAC verification sits in the anti-corruption layer where
provider knowledge belongs; error-dialect translation does too; the ledger's attack surface drops
to zero public paths; and swapping or adding a provider genuinely touches one service — which is
live, since CasinoApiPro has no table games at any tier.

Timeout budget: Integration → Orchestrator 4 s, Orchestrator → Wallet 3 s. See ADR 0008 for the
full resilience policy and the rule that an infrastructure failure must surface as a 5xx, never as
a mapped business error.

The rest of this ADR — the five-service set, the Player replica, schema-per-service, the
Orchestrator's zero-invariants rule, and the rejected alternatives — stands unchanged.

## Context

Koumarin is deliberately distributed beyond what the workload requires: there is no independent
scaling axis, no second team, and no load. That was raised and accepted — the architecture is part
of the deliverable (a .NET / Linux / Kubernetes / RabbitMQ portfolio piece).

Accepting that, the decomposition still has to respect one hard constraint. As a **seamless
wallet**, our response body carries the player's balance inside a 6-second provider timeout. Any
service placed on that path adds latency and becomes a second thing that must be up for a bet to
succeed.

The initial proposal was four services — Wallet, Orchestrator, Integration, Player. An earlier
counter-proposal was three, folding Player into Wallet. This ADR records the resolution.

## Decision

### Five services

| Service | Owns | Schema | Ingress |
|---|---|---|---|
| **Wallet** | Ledger, accounts, idempotency, outbox, HMAC verification, player-status replica | `wallet` | **Public — the only one** |
| **Player** | Player identity, registration, status, currency. System of record. | `player` | Private |
| **Integration** | All CasinoApiPro knowledge: token exchange + cache, catalogue sync, session/launch minting | `integration` | Private |
| **Orchestrator (BFF)** | Player-facing API, use-case coordination, read-model serving | — | Private, behind auth |
| **Reporting/Recon** | Projections, history, reconciliation vs `/v1/transactions` | `reporting` | Private |

### Constraint 1 — two inbound paths that are never merged

- **Provider path (money):** CasinoApiPro → **Wallet, directly.** No gateway, no orchestrator, no
  aggregation layer.
- **Player path:** Browser → Orchestrator → Player / Integration.

Routing the provider callbacks through the Orchestrator or an API gateway "for consistency" is the
single most damaging change available to this design, and is prohibited.

### Constraint 2 — Wallet never calls Player synchronously

Player is the system of record for identity and status. **Wallet keeps a local read-model replica**
of `(player_id, status, currency)`, maintained from `player.*` events (ADR 0005). That replica —
not Player — is authoritative for answering `player_blocked` and `player_not_found` on a callback.

This is what makes the Player split safe, and it is what makes RabbitMQ load-bearing in this
system rather than decorative.

**Accepted trade-off:** blocking a player is eventually consistent, typically sub-second. A player
blocked at T may place a bet at T+200ms. Acceptable for play money, and one of the concrete reasons
real money remains a locked-out constraint — it would require synchronous authorisation and
therefore a different decomposition.

The replica stores `last_event_at` so staleness is observable rather than invisible.

### Constraint 3 — the Orchestrator holds zero business invariants

Services named "orchestrator" tend to accrete business rules until they become a god service
surrounded by anemic domains — the exact failure ADR 0001 exists to prevent.

Its legitimate job is coordinating multi-step player-facing use cases: *launch a game* = validate
the player → request a session from Integration → return the launch URL to the client.

**The rule:** if a rule can be *violated*, it belongs in a domain core. If it is merely a
*sequence*, it belongs in the Orchestrator. The Orchestrator has an Application layer and
essentially no Domain. Enforced by review, and by an architecture test asserting it declares no
aggregates.

### Constraint 4 — no service reads another service's tables

Schema-per-service on one Postgres instance, each service with a Postgres role scoped to its own
schema (granted by migration — ADR 0003). Wallet's role cannot read `player`; the replica exists
precisely so it does not need to. Cross-service data arrives as events or via an API call, never a
join.

### Integration is the only service that knows CasinoApiPro exists

It is an anti-corruption layer: provider field names, error-code dialects, token audiences, and
launch-URL semantics stop there. Adding or swapping a provider changes one service.

## Consequences

- **Positive:** the money path involves exactly one service and one database transaction. Wallet
  can serve bets while Player, Integration, Orchestrator, and Reporting are all down.
- **Positive:** the provider boundary is isolated, so a second provider is additive. This is
  immediately relevant — CasinoApiPro's catalogue has no table games (no blackjack, no roulette) at
  any tier, so adding a second provider for table games is a live possibility, and it is a
  one-service change.
- **Positive:** one public ingress makes the security surface trivially auditable (ADR: see
  architecture doc § Security). Everything else is default-deny NetworkPolicy.
- **Cost accepted:** five deployments, five pipelines, five sets of migrations, and distributed
  tracing becomes necessary rather than optional. Bought knowingly.
- **Cost accepted:** eventual consistency on player status, as described above.
- **Discipline required:** the Orchestrator constraint and the no-cross-schema-reads constraint are
  both easy to violate under deadline pressure and both degrade the design permanently. The
  Postgres grants make the second one structurally impossible, which is why they are migrations
  rather than documentation.
- **Reporting is independently replayable.** Because it owns only projections derived from events,
  it can be rebuilt from the ledger without touching player-facing traffic.

## Alternatives considered

- **Modular monolith.** One solution, the same DDD layering, strict project boundaries, one deploy,
  one Postgres, RabbitMQ retained only for the outbox→projection path. It satisfies every
  functional requirement, is far simpler to operate, and leaves every boundary as a clean
  extraction seam. **Rejected only because the distributed architecture is itself a deliverable.**
  Recorded prominently: this remains the correct answer if the goal ever becomes shipping rather
  than demonstrating.
- **Three services, Player folded into Wallet** (proposed earlier the same day). Avoids the
  eventual-consistency window by making status a local table with no replication. Superseded: the
  read-model replica achieves the same hot-path guarantee while keeping identity in its own
  service, so the split no longer carries the risk that motivated folding it in.
- **Wallet calls Player synchronously on each debit.** The intuitive reading of a Player service.
  Rejected: two services must then be up for any bet, and a network hop is added inside the
  6-second budget — for a check that changes state rarely and is a perfect fit for replication.
- **An API gateway in front of everything, including the callbacks.** Conventional and tidy.
  Rejected: latency on the money path for no benefit, since there is exactly one external caller
  and it authenticates by HMAC rather than by anything a gateway would validate.
- **Reporting folded into the Orchestrator** (both serve read models, so the overlap is real).
  Rejected on operational grounds: a reconciliation backlog or an event-replay would starve
  player-facing requests sharing the same process.
- **Separate Postgres instance per service.** Purer isolation. Rejected on cost for a demo;
  schema-per-service with scoped roles gives the ownership guarantee that actually matters, and
  splitting instances later is a migration rather than a redesign.
