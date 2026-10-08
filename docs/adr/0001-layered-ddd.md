# ADR 0001 — Layered DDD: Core → Application → Infrastructure

- **Status:** Accepted
- **Date:** 2026-10-08
- **Deciders:** Owner (Mario), Claude
- **Scope:** The internal structure of every Koumarin service. Records the decision made on
  2026-10-08 that all business logic and checks live in the domain core, with an application layer
  and an infrastructure layer around it.

## Context

Koumarin is a seamless-wallet operator: CasinoApiPro calls our wallet for every bet and win, and
our response body carries the player's balance. The rules that protect money — a balance may not go
negative, a `transactionId` is applied at most once, a refund must reference a real prior
transaction — are the most valuable code in the system. They must not end up scattered across
controllers, SQL strings, and message handlers.

Two further forces shape the structure:

- **TDD is required** (ADR 0007). Tests of business rules must run without a database, a broker, or
  an HTTP host, or the red-green loop becomes too slow to actually practise.
- **Five services** (ADR 0006) need one predictable shape, so moving between them costs nothing.

## Decision

Every service is four projects, with references pointing **inward only**:

```
Koumarin.<Service>.Api             → Infrastructure → Application → Domain
Koumarin.<Service>.Infrastructure  → Application → Domain
Koumarin.<Service>.Application     → Domain
Koumarin.<Service>.Domain          → (nothing)
```

1. **Domain** — aggregates, entities, value objects (including `Money`), domain events, and
   repository *interfaces*. All invariants live here. **Zero dependencies**: no EF Core, no Dapper,
   no ASP.NET, no `IConfiguration`, no logging framework. Only the BCL.
2. **Application** — use cases / handlers, one per operation (`DebitHandler`, `CreditHandler`,
   `LaunchGameHandler`). Owns the **transaction boundary** and orchestrates: load aggregate, call
   domain, persist, enqueue outbox. It coordinates; it does not decide.
3. **Infrastructure** — Dapper repositories, EF Core read contexts, Postgres, Redis, RabbitMQ
   publishing, SQL migrations. Implements the interfaces the Domain declares.
4. **Api** — ASP.NET Core host, HMAC middleware, endpoint definitions, DI composition root.

**Supporting rules:**

- **Domain types carry no persistence concerns** — no ORM attributes, no column names, no
  `[Key]`. Mapping lives entirely in Infrastructure, which ADR 0002's hand-written SQL makes
  natural.
- **The Application layer owns the transaction**, not the repository. A single wallet callback must
  commit a ledger row, a balance update, and an outbox row together (ADR 0004, ADR 0005); a
  repository that opened its own transaction per call would make that impossible.
- **The dependency rule is enforced, not aspirational.** An architecture test (NetArchTest or
  equivalent) fails the build if Domain acquires an outward reference. Without enforcement this
  decision decays within weeks.
- **The Orchestrator service holds no invariants at all** (ADR 0006) — it has an Application layer
  and essentially no Domain. That is correct for it and wrong for everything else.

## Consequences

- **Positive:** business rules are in one place per service, readable without knowing anything
  about SQL or HTTP. Domain tests need no Docker and run in milliseconds, which is what makes TDD
  practical rather than theoretical (ADR 0007).
- **Positive:** swapping persistence is a change in one layer. This is not hypothetical — ADR 0002
  puts two different data-access libraries behind the same interfaces.
- **Cost accepted:** roughly twenty projects across five services. More solution noise, slower
  cold builds, more `.csproj` wiring. For a system whose core asset is its money rules, the
  explicitness is worth it.
- **Cost accepted:** no change tracking means repositories do explicit reads and explicit writes.
  More code than `SaveChanges()`, and on the ledger path that visibility is the point.
- **Discipline required:** the easiest way to break this is a "small" validation in a controller or
  a rule expressed in a `WHERE` clause. The architecture test catches reference violations; it
  cannot catch a rule written in the wrong layer. That is a review responsibility.

## Alternatives considered

- **One project per service**, folders instead of projects. Simpler to set up, and the dependency
  rule becomes unenforceable — nothing stops a domain class from `using` Dapper. Rejected: the rule
  that cannot be checked is the rule that erodes.
- **Vertical slice architecture** (feature folders, each slice owning its own read/write path).
  Genuinely good for CRUD-heavy apps and increasingly popular in .NET. Rejected because the
  valuable logic here is concentrated in a few aggregates reused across many operations, which is
  precisely where a shared domain core beats per-feature slices — and because layered DDD was the
  explicit requirement.
- **Hexagonal / ports-and-adapters naming** (`Core`, `Ports`, `Adapters`). The same structure with
  different labels. Rejected only to keep the vocabulary conventional for .NET readers.
- **Anemic domain + service classes** — entities as data bags, logic in `WalletService`. Rejected
  outright: it is the specific failure mode this ADR exists to prevent.
