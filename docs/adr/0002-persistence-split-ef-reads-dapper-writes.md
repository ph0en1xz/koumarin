# ADR 0002 — Persistence split: EF Core for reads, Dapper for writes

- **Status:** Accepted
- **Date:** 2026-10-08
- **Deciders:** Owner (Mario), Claude
- **Scope:** How every Koumarin service reads from and writes to Postgres. Records the decision
  made on 2026-10-08. Has two direct consequences carried in ADR 0003 (migrations) and ADR 0005
  (outbox).

## Context

The wallet write path is the most constrained code in the system. A single `POST /wallet/debit`
must, inside one transaction:

- take a row lock on the player's account to serialise concurrent bets (`SELECT … FOR UPDATE`),
- insert a ledger row whose `transaction_id` carries a unique constraint, and treat the unique
  violation as a legitimate control-flow branch (ADR 0004),
- update the cached balance, and insert an outbox row (ADR 0005),
- all within a latency budget set by a 6-second provider timeout.

Lock ordering, the exact statement issued, and the failure mode of a constraint violation are all
load-bearing here. An ORM change tracker sits between the code and those behaviours: the SQL that
actually runs, and when, becomes an inference from entity state rather than something visible in
review.

Conversely, Reporting/Recon runs richer queries — projections, joins, history for a player,
reconciliation sweeps against the provider's `/v1/transactions`. That is where LINQ and a mapped
object graph genuinely reduce code.

## Decision

1. **All writes go through Dapper**, with hand-written SQL. No exceptions — this includes the
   ledger, the outbox, the player-status replica, and every other table in the system.
2. **EF Core is read-only.** It is never used to insert, update, or delete, and no `SaveChanges()`
   is called anywhere in the solution.
3. **EF Core is scoped to the Reporting/Recon service.** Wallet, Player, Integration, and
   Orchestrator use Dapper for both reads and writes. Wallet's only read on the hot path is a
   single-row balance lookup by `player_id`, where an EF `DbContext` would add a second mental
   model and a mapping configuration to earn nothing.
4. **Domain types stay free of persistence concerns** (ADR 0001) — no ORM attributes anywhere.
   Mapping lives in Infrastructure, which hand-written SQL makes natural rather than awkward.
5. **EF Core read contexts map to read models**, not to write aggregates. They are allowed to map
   to views and projections. They must never be treated as the schema's source of truth
   (ADR 0003).

## Consequences

- **Positive:** the SQL on the money path is visible in the repository, reviewable line by line,
  and diffable. `FOR UPDATE`, `ON CONFLICT`, and the lock ordering are explicit instead of emergent.
- **Positive:** Reporting keeps LINQ ergonomics where query complexity actually warrants them.
- **Consequence → ADR 0003:** with no EF Core write model, EF migrations would have to be generated
  from a model maintained purely to emit DDL. Schema ownership moves to hand-written SQL
  migrations.
- **Consequence → ADR 0005:** MassTransit's EF Core transactional outbox requires EF Core to
  enlist in the write transaction, which this decision rules out. The outbox is hand-rolled.
- **Cost accepted:** two data-access libraries, two mental models. Mitigated by scoping them by
  service and layer rather than by developer preference: *if it writes, it is Dapper; if it is
  Reporting, it may read with EF Core.* Any other use is a review failure.
- **Risk:** EF Core read contexts can drift from the real schema, since nothing generates them from
  it. Mitigated by integration tests that run the real migrations against a Testcontainers Postgres
  and then query through EF Core (ADR 0007) — drift fails a test rather than a production query.
- **No unit of work.** Each handler explicitly opens a transaction, passes the connection to its
  repositories, and commits. More ceremony than `SaveChanges()`; on the ledger path the ceremony is
  the feature.

## Alternatives considered

- **EF Core for everything.** The conventional .NET default, one library, migrations included.
  Rejected: it puts the change tracker between the code and the lock/constraint behaviour on the
  one path where that behaviour is the whole design. `FOR UPDATE` and `ON CONFLICT` are expressible
  via raw SQL in EF Core, but then the ORM is being bypassed exactly where it matters while still
  being carried everywhere else.
- **Dapper for everything, no EF Core at all.** A genuinely strong option — it removes the
  two-library cost entirely, and Dapper reads are perfectly capable. Rejected because Reporting's
  projection and join-heavy queries are where an ORM pays, and because the read/write split was the
  stated requirement. Recorded honestly: if the two-library cost ever outweighs Reporting's
  convenience, collapsing to Dapper-only is the cheap simplification, not a rewrite.
- **Dapper for reads, EF Core for writes** — the conventional inversion of this ADR, trading
  explicit SQL on queries for a unit of work on writes. Rejected: it optimises the wrong path. The
  queries are simple and the writes are the dangerous part.
- **A repository abstraction hiding which library is used.** Rejected as false economy: the whole
  value of Dapper here is that the SQL is visible. An abstraction designed to conceal it defeats
  the decision.
