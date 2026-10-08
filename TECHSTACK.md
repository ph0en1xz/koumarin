# Koumarin — Tech Stack Reference

> This file is the authoritative, fast-load tech stack reference for the Koumarin project.
> Claude agents and skills: read this file FIRST. Then read `CLAUDE.md` for product context.

**Status:** Pre-architecture. Repo scaffolded 2026-10-08. No code yet. The service decomposition
below is the agreed shape; the architecture session has not run, so anything marked **OPEN** is
undecided and must not be assumed.

---

## What this is
A **play-money** online casino demo. Games are supplied by **CasinoApiPro**; Koumarin is the
*operator* — it owns the player wallet, the ledger, and the player relationship. No real money
enters or leaves the system. No gaming licence, KYC, or payment rails in scope.

The architecture is deliberately heavier than a play-money demo requires: this is also a
.NET / Linux / Kubernetes / RabbitMQ portfolio piece, and the architecture is part of the
deliverable. That trade-off was made knowingly (2026-10-08) — do not "simplify" it away.

## Platform — Linux end to end (LOCKED)
Development, build, container images, and deployment are **all Linux**. There is no Windows
target at any stage.

- **Repo lives in WSL:** `/home/mariosl/repos/koumarin` on **Ubuntu 22.04.3 LTS**, user `mariosl`.
- From Windows, reach it at `\\wsl.localhost\Ubuntu\home\mariosl\repos\koumarin`.
- **Never place or clone this repo under `/mnt/c`.** Crossing the DrvFs boundary costs an order of
  magnitude on file I/O and breaks `inotify`, so `dotnet watch` and test watchers silently stop
  firing. ext4 inside WSL only — same arrangement as `a-game`.
- **Runtime identifier:** `linux-x64` (`linux-musl-x64` only if an Alpine base is chosen).
- **Line endings: LF everywhere.** Enforced by `.gitattributes` and `core.autocrlf=false`.
  This matters because the repo is edited from a Windows IDE while living on the Linux filesystem.
- All shell scripts, Dockerfiles, and CI steps assume a POSIX shell. No PowerShell in the repo.

### Toolchain state inside WSL (Wave 0 complete, 2026-10-08)
| Tool | Version | Notes |
|---|---|---|
| .NET SDK | **10.0.401** | `~/.dotnet`, installed via `dotnet-install.sh` — no sudo, home-dir install |
| helm | **3.22.0** | `~/.local/bin`, official `get-helm-3` installer with `USE_SUDO=false` |
| Docker | 28.5.1 | WSL daemon — also what Testcontainers uses |
| kubectl | installed | — |
| git | 2.34.1 | identity set; `gh` authenticated as `ph0en1xz` |

**Shell-config gotcha, recorded because it cost a debugging cycle:** `DOTNET_ROOT` and the PATH
entry live in **`~/.profile`**, not `~/.bashrc`. Ubuntu's `.bashrc` returns early for
non-interactive shells, so exports placed there are invisible to scripts, CI steps and
`bash -lc` — `dotnet` resolves interactively and then mysteriously does not resolve anywhere else.
Environment variables belong in `.profile`; `.bashrc` is for interactive shell behaviour.

## Runtime & Language
- **Framework:** .NET **10** (LTS, released 2025-11-11, supported to **2028-11-14**), ASP.NET Core.
- **Language:** C#, `<Nullable>enable</Nullable>` + `<TreatWarningsAsErrors>` repo-wide via
  `Directory.Build.props`. Central package versions via `Directory.Packages.props`.
- **⚠️ Do NOT build on .NET 8 or 9** — both reach end of support **2026-11-10**.
  .NET 11 (LTS) is expected November 2026 — revisit the pin then, do not adopt an RC.
  Source: https://dotnet.microsoft.com/en-us/platform/support/policy/dotnet-core

## Repo Layout
- **Monorepo.** All services, IaC, deploy manifests, and docs in one repo — one Argo CD
  app-of-apps, one CI matrix, atomic cross-service changes.
- GitHub remote: to be created under **`ph0en1xz`** (not yet created as of 2026-10-08).
- ADRs live here in `docs/adr/`. The living architecture doc lives in the home base at
  `documentation/architecture/koumarin/architecture.md` (that home base is on Windows —
  this is the one intentional cross-boundary path).

## Service Decomposition (5 services — designed 2026-10-08)
Full design: home base `documentation/architecture/koumarin/architecture.md`.

| Service | Owns | Ingress |
|---|---|---|
| **Integration** | **ALL** CasinoApiPro knowledge, both directions: the 5 inbound callbacks, HMAC verification, payload + error translation, DTOs, provider resilience (ADR 0008), token cache, catalogue sync, session minting | **Public — the only machine-facing one** |
| **Orchestrator** | Sole coordinator: provider settlement routing + player-facing use cases | Public, user-authenticated (SPA only) |
| **Wallet** | Postgres ledger, idempotency, outbox, player-status replica | **None — internal only** |
| **Player** | Player identity, registration, status, currency. System of record. | None — internal |
| **Reporting/Recon** | Ledger projections, history, reconciliation vs `/v1/transactions`. Pure async consumer. | None — internal |

### The four structural rules (ADR 0006 §Amendment, 2026-10-08)
1. **Integration is the only service that touches CasinoApiPro, in both directions.** Every
   inbound callback and every outbound call terminates there. Provider HMAC, field names, error
   dialects and DTOs stop at that boundary and never travel inward.
2. **Wallet has no public ingress.** Internal-only, reachable from the Orchestrator alone. The
   internet cannot address the ledger.
3. **The Orchestrator is the only coordinator** — settlement and player flows alike — and holds
   **ZERO business invariants**. Routing a `balance` callback to a read and a `debit` to a write is
   *sequence*; whether the player can afford it, is blocked, or is a duplicate is a *rule* and
   stays in Wallet's domain. Enforced by an architecture test asserting it declares no aggregates.
4. **Idempotency on `transaction_id` makes every hop safely retryable** — internal hops included.

**Settlement path:** `CasinoApiPro → Integration → Orchestrator → Wallet`
**Player path:** `Browser → Orchestrator → Player / Integration`

**Timeout budget** (inside the provider's 6 s): Integration → Orchestrator **4 s**,
Orchestrator → Wallet **3 s**. One internal retry permitted; `transaction_id` makes it safe.

**Accepted cost:** three services must be up to settle a bet. Latency is irrelevant (two
in-cluster hops vs a 6,000 ms budget) but **availability compounds** — so Integration, Orchestrator
and Wallet all run multiple replicas with pod disruption budgets, and none of them may touch
RabbitMQ, Player, or catalogue work while a callback is in flight.

**Player is separate, but Wallet never calls it synchronously.** Wallet keeps a local read-model
replica of `(player_id, status, currency)` fed by `player.*` events; that replica is authoritative
for the settlement decision. Accepted trade-off: blocking a player is eventually consistent
(sub-second) — fine for play money, and one of the reasons real money stays locked out.

## Messaging
- **Broker:** RabbitMQ.
- **RabbitMQ is never in the money path.** CasinoApiPro waits on Koumarin's HTTP response body
  for the new balance (6s timeout, 2 retries) — that cannot be served from a queue.
- **Pattern: transactional outbox.** The money movement, the `transactionId`, and the outbox row
  are written in ONE Postgres transaction; a relay publishes after commit. No dual write, no lost
  events, no added latency on the callback.
- Queue carries: ledger events → projections/reconciliation, catalogue sync fanout, and a
  dead-letter queue for failed reversals needing manual review.
- **The outbox is hand-rolled** (decided 2026-10-08). MassTransit's EF Core outbox needs EF Core
  inside the write transaction, which the persistence split rules out. So: `INSERT` into `outbox`
  in the same Dapper transaction as the ledger row and balance update, with a separate relay
  publishing after commit. Three inserts, one transaction, full control of the SQL.
- RabbitMQ access via plain `RabbitMQ.Client`, or MassTransit used purely as a transport — not for
  its outbox. Topic exchange `koumarin.events`, quorum queues, a DLQ per consumer queue, and
  idempotent consumers (inbox dedupe table).

## Database & Cache
- **Ledger:** PostgreSQL — authoritative, the single consistency boundary for money.
  Unique index on `transactionId`; movement + id recorded in one transaction.
- **Cache:** Redis — CasinoApiPro access-token cache (tokens live 3600s; cache them, never mint
  per request) and HMAC timestamp replay/freshness tracking.
- **Postgres: AWS RDS (DECIDED 2026-10-08).** Managed, so backups, PITR and failover are not
  hand-rolled for the ledger. Lives in private subnets; reachable only from the EKS node security
  group. Single instance for a demo — Multi-AZ is a flag to flip, not a redesign.
- **Redis: in-cluster (DECIDED 2026-10-08).** It holds only the token cache and the HMAC replay
  window, both rebuildable, so losing the instance costs a cold cache rather than data —
  ElastiCache would be paying for durability nothing here needs. *Caveat:* an emptied replay set
  briefly narrows HMAC replay protection to the timestamp-freshness check alone, so do not move
  anything non-rebuildable into Redis later without revisiting this.
- **Schema layout:** schema-per-service on one Postgres instance (`wallet.*`, `player.*`,
  `integration.*`, `reporting.*`). **No service reads another service's tables** — each gets a
  Postgres role scoped to its own schema. Wallet's player-status replica arrives as events,
  never as a cross-schema join.

## Data Access — EF Core reads, Dapper writes (DECIDED 2026-10-08)
- **All writes go through Dapper**, hand-written SQL. The wallet path needs `SELECT … FOR UPDATE`,
  `INSERT … ON CONFLICT`, deliberate lock ordering, and unique-violation handling — explicit SQL
  makes that reviewable, where EF Core's change tracker would put indirection on the one path
  where that is least acceptable.
- **EF Core is read-only**, and scoped to **Reporting/Recon**, where richer projections and joins
  pay for a `DbContext`. **Wallet is pure Dapper** — its only read is a single-row balance lookup,
  where EF Core would add a second mental model to earn nothing.
- **Migrations: hand-written SQL via DbUp or FluentMigrator.** With no EF Core write model, EF
  migrations would be generated from a phantom model maintained only to emit DDL, and would drift.
  Hand-written migrations match hand-written queries — and the DDL creating the ledger's unique
  index stays visible in review.

## Domain Model — DDD + TDD
Every service uses the same four-project layering, dependencies pointing **inward only**:

```
Koumarin.<Service>.Domain          ← aggregates, value objects, domain events, repository
                                     INTERFACES, invariants.  ZERO dependencies.
Koumarin.<Service>.Application     ← use cases/handlers, transaction boundary, ports
Koumarin.<Service>.Infrastructure  ← Dapper repos, EF Core read contexts, Redis, RabbitMQ, migrations
Koumarin.<Service>.Api             ← ASP.NET host, HMAC middleware, endpoints, DI composition
```

- `Api → Infrastructure → Application → Domain`. **Domain references nothing** — no EF, no Dapper,
  no ASP.NET. Enforced by an architecture test (NetArchTest or equivalent) that fails the build.
- **All business logic and checks live in Domain.** The Application layer orchestrates and owns the
  transaction boundary; it does not decide.
- **`Money` is a value object** — `decimal` amount + ISO 4217 currency, arithmetic that refuses to
  mix currencies, no public setters. Shared via `Koumarin.Shared.Money`. Trade-off accepted: a
  shared primitive couples the domains slightly, but guarantees identical rounding between the
  ledger and reporting, which matters more.
- **TDD, domain-first.** Domain tests need no Docker, no DB, no host, so they run in milliseconds —
  build the core red-green-refactor, then Application, then Infrastructure with Testcontainers.
  The layering is what makes TDD fast here rather than theoretical.
- **Idempotency spans two layers by design:** the Domain *states* the rule (a `transactionId` is
  applied at most once), the **unique index enforces it atomically** under concurrency, and the
  Application layer translates a unique-violation into the idempotent replay response. This is
  deliberate, not a leak — do not "fix" it by moving enforcement into the aggregate.

## Money representation (NON-NEGOTIABLE)
- `decimal` in C#. **Never** `double`/`float` on any monetary path.
- CasinoApiPro sends and expects amounts as **decimal strings** (`"10.50"`, not `10.5`) to avoid
  IEEE-754 error. Custom `System.Text.Json` converters at the boundary.

## External Integration — CasinoApiPro
- **Integration model: SEAMLESS WALLET (LOCKED).** CasinoApiPro calls Koumarin's wallet API in
  real time for every bet and win; Koumarin holds the single authoritative balance and returns it
  in the response body. There is **no transfer wallet** — no pre-funding a provider-side balance,
  no deposit/withdraw into the game provider, no periodic balance sync job, and no second place
  where a balance lives.
  - *Why this is the whole design:* seamless is what puts our wallet on the provider's critical
    path, which is what makes idempotency, atomicity, and the latency budget non-negotiable. A
    transfer wallet would move those concerns out of the request path entirely and make most of
    this architecture unnecessary.
- **Resilience and DTOs live in Integration (ADR 0008).** `IHttpClientFactory` + a Polly v8
  pipeline per endpoint. **Outbound:** retry with jitter, per-endpoint timeouts, circuit breaker
  per endpoint group; `POST /v1/sessions` is **not idempotent**, so at most one retry there.
  Breaker-open degrades (stale catalogue served, session minting fails fast) rather than dying.
  **Inbound: no circuit breaker** — CasinoApiPro already retries, and failing fast with a mapped
  business error would make a transient fault permanent. Provider DTOs are declared in Integration
  and never leave it; mapping is explicit or source-generated, never reflection-based.
- **Base:** `https://api.casinoapipro.com/v1` (HTTPS only).
- **Auth:** `POST /v1/auth/token` with `api_key`/`api_secret` → bearer token, `expires_in` 3600.
  Three token audiences: `operator-api`, `game-launcher`, `dashboard`.
- **Launch:** `POST /v1/sessions` (`game_id`, `player_id`, `currency`) → short-lived `launch_url`,
  scoped to one operator/player/game. Mint fresh per launch; embed in an `<iframe>`.
- **Inbound callbacks Koumarin must expose** (base URL configurable):
  `POST {base}/wallet/balance` · `/debit` · `/credit` · `/refund` · `/rollback`
- **Signature:** `x-signature = hex(HMAC_SHA256(secret, "{timestamp}.{raw_body}"))`, lowercase
  hex, literal dot, with `x-timestamp`. Freshness validation is the operator's job (~5 min window).
  **Read the raw body bytes before model binding** — ASP.NET Core will otherwise re-serialize and
  break the signature.
- **Required guarantees:** idempotency on `transactionId` (replay returns the same balance +
  `duplicate: true`), atomicity, and speed (6s timeout, 2 retries, exponential backoff — a timeout
  is ambiguous, the money may have moved).
- **Error codes:** `insufficient_funds`, `player_blocked`, `player_not_found`,
  `currency_mismatch`, `transaction_not_found`, `already_reversed`, `invalid_request`.
  Unmapped codes = terminal `502 WALLET_REJECTED`, no retry.
- **Plan:** sandbox only, indefinitely. Free, never expires, play-money wallet with real behaviour
  (balance checks, duplicate rejection, injectable failures). Paid tiers start $499/mo + 2–5% GGR —
  not needed. Catalogue is 14 proprietary games (1 crash, 11 instant, 3 slots).
- Docs: https://casinoapipro.com/docs/introduction · spec: https://api.casinoapipro.com/openapi.json

## Hosting & Deployment
- **Target:** AWS EKS, Linux nodes. Design for this target from the first draft, not for local.
- **Packaging: Helm (DECIDED 2026-10-08).** One chart per service under `deploy/charts/`, plus an
  umbrella chart; one values file per environment. Helm is also what the RabbitMQ Cluster Operator
  and External Secrets Operator ship as, so there is one packaging tool rather than two.
- **OPEN:** Argo CD GitOps (strongly expected, mirroring the a-game setup) — confirm app-of-apps
  layout when the deploy work starts.
- RabbitMQ via the RabbitMQ Cluster Operator; secrets via External Secrets Operator → SSM.
- **Public surface:** the 5 wallet callback paths are the **only** justified public ingress —
  CasinoApiPro must reach them from the internet. Everything else stays private. HMAC always on,
  plus an IP allowlist if CasinoApiPro publishes egress ranges.

## Infrastructure as Code
- **Tool:** Terraform (home-base standard; remote state in S3 + lock). Apply only after plan review.

## CI/CD
- **Platform:** GitHub Actions, `ubuntu-latest` runners. Build once, promote the same artifact.
  Never manual prod changes.
- **OPEN:** environment set. Likely dev → prod only for a demo; confirm in architecture.

## Secrets
- **Manager:** AWS SSM Parameter Store / Secrets Manager, synced to Kubernetes via External Secrets.
- **In secrets:** CasinoApiPro `api_key` + `api_secret`, the HMAC callback secret, Postgres and
  Redis credentials, RabbitMQ credentials.
- **Never in repo.** `.env` is local-only and gitignored.

## Linting & Quality
- **Format:** `dotnet format` + Roslyn analyzers. Warnings are errors.
- **Tests:** xUnit via `dotnet test`. Integration via `WebApplicationFactory<Program>`.
  DB isolation via **Testcontainers (`Testcontainers.PostgreSql`) against real Postgres** —
  **never the EF Core InMemory provider**, which enforces neither unique indexes nor transactions
  and would pass the exact idempotency tests production rejects. Testcontainers uses the WSL
  Docker daemon directly.
- Inject `TimeProvider`; never call `DateTime.UtcNow` directly.

## Observability
- **OPEN** — not yet decided. Tracing across RabbitMQ hops is where distributed debugging
  actually hurts, so decide alongside the first deployment. CasinoApiPro exposes `/metrics`
  (Prometheus) on their side.

## Key Commands
All commands run **inside WSL**, from `/home/mariosl/repos/koumarin`.
```bash
dotnet build                          # build the solution
dotnet test                           # run tests (Testcontainers needs the Docker daemon up)
dotnet format --verify-no-changes     # format check (CI gate)
terraform -chdir=infra plan           # IaC plan (review before apply)
terraform -chdir=infra apply          # IaC apply (only after plan review)
kubectl get pods -n koumarin          # read-only diagnostics
```

## Hard Constraints (LOCKED — do not deviate without explicit approval)
- **Linux only**, end to end: dev in WSL, Linux container images, Linux EKS nodes. The repo lives
  on ext4 inside WSL and never under `/mnt/c`.
- **Play money only.** No real-money deposits or withdrawals, no KYC, no payment rails.
  Introducing any of these is a licensing decision, not an engineering one.
- **.NET 10** — not 8, not 9 (both EOL 2026-11-10), not a .NET 11 RC.
- **Integration is the ONLY service that touches CasinoApiPro**, inbound and outbound. No provider
  DTO, field name, error code, or HMAC concern may appear in any other service.
- **Wallet is never publicly reachable.** Settlement reaches it only via Orchestrator.
- **An infrastructure failure on the settlement path returns a 5xx, never a mapped business
  error.** Their contract treats unmapped/`invalid_request` codes as terminal with no retry, so a
  wrong code converts a retryable blip into a permanently lost settlement (ADR 0008).
- **RabbitMQ never in the wallet callback path.** Outbox after commit, never before the response.
- **Money is `decimal`**, serialized as a decimal string at the CasinoApiPro boundary.
- **The ledger is one consistency boundary** — never split the wallet's money movement across a
  network call. Idempotency on `transactionId` + atomicity are not optional.
- **CasinoApiPro sandbox only** — no paid tier, no live mode.
- Monorepo — do not split into repo-per-service without an ADR.
