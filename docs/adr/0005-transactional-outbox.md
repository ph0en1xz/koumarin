# ADR 0005 — Transactional outbox; RabbitMQ stays out of the money path

- **Status:** Accepted
- **Date:** 2026-10-08
- **Deciders:** Owner (Mario), Claude
- **Scope:** How Koumarin publishes events to RabbitMQ without risking the money path. Pairs with
  ADR 0004; constrained by ADR 0002.

## Context

Koumarin is event-driven: Reporting builds projections from ledger events, Wallet maintains its
player-status replica from `player.*` events (ADR 0006), and reconciliation consumes ledger history.

But the wallet callback path cannot involve RabbitMQ at all. CasinoApiPro waits on our **HTTP
response body** for the new balance, inside a 6-second timeout. A balance cannot be served from a
queue, and a broker hop cannot be added to a request that must return a computed value.

That leaves the classic dual-write problem. Writing to Postgres and publishing to RabbitMQ are two
systems with no shared transaction, so any naive ordering loses:

- **Publish before commit:** the transaction can still fail. Consumers then see a bet that never
  happened — unrecoverable, because the event is already out.
- **Publish after commit:** the process can die between commit and publish. The money moved and no
  event exists — a permanently missing projection, with nothing to detect it.

Neither is acceptable for a ledger.

ADR 0002 removes the obvious framework answer: MassTransit's transactional outbox requires EF Core
to enlist in the write transaction, and all writes here go through Dapper.

## Decision

1. **Hand-rolled transactional outbox.** The event is written as a row in an `outbox` table **in
   the same Dapper transaction** as the ledger row and the balance update (ADR 0004). Three
   inserts, one commit, one atomic unit. If the transaction rolls back, the event never existed.
2. **A relay publishes after commit, off the request path.** A background service in the owning
   service polls for rows where `published_at IS NULL`, publishes to RabbitMQ, and stamps
   `published_at`. The HTTP response to CasinoApiPro does not wait for it.
3. **Indexed for the relay:** a partial index on `(published_at) WHERE published_at IS NULL`, so
   the poll cost is proportional to the backlog, not to the table.
4. **Delivery is at-least-once.** A crash between publishing and stamping republishes. Therefore
   **every consumer must be idempotent**, backed by an inbox dedupe table keyed on event id. This
   is a requirement on consumers, not an aspiration.
5. **Ordering is per-player best-effort, not guaranteed.** Consumers must tolerate out-of-order
   arrival. Where order matters (the player-status replica), events carry a monotonic
   `occurred_at`/version and the consumer ignores anything older than what it has already applied.
6. **Topology:** topic exchange `koumarin.events`, one durable queue per consumer, **quorum
   queues**, and a dead-letter queue per consumer queue. Routing keys are
   `ledger.debited`, `ledger.credited`, `ledger.refunded`, `ledger.rolled_back`, `player.blocked`,
   `player.unblocked`, `player.registered`, `catalogue.updated`.
7. **RabbitMQ client:** plain `RabbitMQ.Client`, or MassTransit used purely as a transport — **not**
   for its outbox. Choosing MassTransit for transport convenience is acceptable; re-introducing its
   outbox is not, because it would require reversing ADR 0002.

## Consequences

- **Positive:** no dual write, no lost events, no phantom events, and **zero added latency** on the
  provider callback. The outbox insert is one more row in a transaction that was already open.
- **Positive:** the broker can be completely down and the casino keeps taking bets. Events queue up
  in Postgres and drain when RabbitMQ returns. For a system whose money path must not depend on a
  broker, this is the main prize.
- **Positive:** the outbox table is an audit log of intent, independently of what the broker did.
- **The relay is a single point of silent failure — the most important operational risk in the
  system.** If it stops, money stays perfectly correct while every projection quietly freezes, and
  nothing errors. **Mitigation is mandatory, not optional:** alert on the age of the oldest
  unpublished outbox row (`published_at IS NULL`), not merely on relay liveness — a running relay
  that is failing to publish looks healthy.
- **Cost accepted:** polling adds latency between commit and publish (sub-second at demo volume).
  `LISTEN`/`NOTIFY` is the documented optimisation if that ever matters; polling is simpler and
  survives a missed notification, which `NOTIFY` does not.
- **Burden on consumers:** at-least-once plus no ordering guarantee means every consumer carries
  dedupe and staleness handling. That is the honest cost of not having distributed transactions.
- **Cleanup needed:** published rows accumulate. A periodic delete of rows published more than N
  days ago, retained long enough to be useful for debugging.

## Alternatives considered

- **Publish inside the transaction.** Rejected: the commit can still fail after a successful
  publish, emitting an event for a bet that did not happen. Worse than a missing event, because it
  is unrecoverable.
- **Publish immediately after commit, no outbox.** The common shortcut. Rejected: a crash in the
  window between commit and publish loses the event permanently and undetectably.
- **MassTransit EF Core outbox.** The framework answer, and the reason MassTransit was originally
  proposed. Ruled out by ADR 0002 — it needs EF Core inside the write transaction. Reversing that
  to gain the outbox would trade explicit SQL on the ledger for a library feature, which is the
  wrong way round.
- **Change data capture (Debezium / Postgres logical replication → Kafka or RabbitMQ).** Genuinely
  robust, requires no application-side outbox, and gives exact ordering. Rejected: a connector, a
  replication slot, and usually Kafka is a large amount of infrastructure to operate for a
  play-money demo, and a stalled replication slot can fill the disk of the database the ledger
  lives on.
- **Two-phase commit across Postgres and RabbitMQ.** Rejected: operationally painful, poorly
  supported, and the outbox pattern exists precisely to avoid needing it.
- **Skip events entirely; let Reporting read the ledger directly.** Simpler, and breaks the
  schema-ownership rule in ADR 0006 (no service reads another's tables) while coupling Reporting to
  Wallet's internal schema. Rejected — though worth noting it would make the system meaningfully
  simpler if the architecture were not itself part of the deliverable.
