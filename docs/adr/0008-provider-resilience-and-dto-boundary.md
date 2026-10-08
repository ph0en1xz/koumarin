# ADR 0008 — Provider resilience and the DTO boundary live in Integration

- **Status:** Accepted
- **Date:** 2026-10-08
- **Deciders:** Owner (Mario), Claude
- **Scope:** Where retries, timeouts, circuit breaking and provider data contracts live. Extends
  ADR 0006's rule that Integration is the only service touching CasinoApiPro.

## Context

ADR 0006 (as amended 2026-10-08) makes Integration the sole provider boundary in both directions.
That boundary is also where every provider-shaped failure appears: a slow `/v1/sessions`, a 500
from `/v1/auth/token`, a TLS reset mid-catalogue-sync, a provider outage.

If resilience policy were spread across callers, each would need its own notion of how many times
to retry CasinoApiPro and when to give up — and the provider's failure modes would leak into
services that are not supposed to know it exists. The same applies to data shape: the provider's
field names are *configurable on their side* (seven request fields and all three response fields
are remappable), so any internal type that mirrors their JSON would be coupled to a contract that
can change without our code changing.

There is also an asymmetry that matters and is easy to miss: **the two directions need opposite
policies.** Outbound, we are the client and may retry freely. Inbound, CasinoApiPro is the client
and *already* retries (6 s timeout, 2 retries, exponential backoff) — so our job there is to be
honest about failure in a way their retry logic understands, not to add resilience of our own.

## Decision

### 1. All provider resilience policy lives in Integration

Implemented with `IHttpClientFactory` plus a Polly v8 resilience pipeline
(`Microsoft.Extensions.Resilience`), configured per provider endpoint. No other service carries a
retry or breaker for anything CasinoApiPro-related, because no other service calls it.

### 2. Outbound policy (Integration → CasinoApiPro)

| Endpoint | Timeout | Retry | Breaker |
|---|---|---|---|
| `POST /v1/auth/token` | 5 s | 3, exponential + jitter | yes |
| `GET /v1/games` | 10 s | 3, exponential + jitter | yes |
| `POST /v1/sessions` | 5 s | **1, cautiously — see below** | yes |

- **Retry with jitter**, never fixed-interval, so retries do not synchronise across replicas.
- **Circuit breaker per endpoint group**, not one global breaker: a failing catalogue sync must not
  stop session minting.
- **`POST /v1/sessions` is not idempotent.** A retry after a lost response may mint a second
  session. The consequence is mild — sessions are short-lived, scoped to one operator/player/game,
  and minting is cheap — but it is a real duplicate, so retries there are limited to one and never
  applied to a response we actually received. Launch URLs are minted fresh per open anyway
  (ADR 0006), which bounds the damage.
- **Token fetches are serialised per replica** so a cold cache under load does not fire a hundred
  concurrent `/v1/auth/token` calls and burn rate limit.
- **Breaker-open behaviour is degraded, not fatal:** a stale cached catalogue is served; session
  minting fails fast with a clear user-facing message. The casino stays browsable when the
  provider is down.

### 3. Inbound policy (CasinoApiPro → Integration → Orchestrator → Wallet)

**No circuit breaker on the settlement path.** This is the deliberate asymmetry.

- **Timeouts only**, per the budget in the architecture doc: Integration → Orchestrator 4 s,
  Orchestrator → Wallet 3 s, inside the provider's 6 s.
- **One internal retry is permitted** on a timeout between our own services, because
  `transaction_id` makes it safe (ADR 0004). Never more than one — the budget does not allow it.
- **An infrastructure failure must surface as a 5xx or a timeout, never as a mapped business
  error.** Their contract treats unmapped and `invalid_request` codes as a terminal
  `502 WALLET_REJECTED` with **no retry**. So returning a business-shaped error for what is
  actually "Wallet is down" would convert a retryable blip into a permanently lost settlement.
  Infrastructure failure → 5xx → they retry → our idempotency makes the retry correct.
- Rationale for no breaker: failing fast inbound buys nothing. CasinoApiPro already backs off and
  retries, and a breaker that returns the wrong code actively destroys settlements. The honest
  response to "Wallet is unreachable" is a 5xx, every time.

### 4. The DTO boundary

- **Provider DTOs are declared in Integration and never leave it.** They mirror CasinoApiPro's wire
  format exactly: decimal strings, their field names, their error codes, their header conventions.
- **Internal contracts are separate types** — settlement commands going inward, domain results
  coming back. Nothing downstream of Integration references a provider DTO, and an architecture
  test asserts the provider DTO namespace is referenced by Integration alone.
- **Mapping is explicit** — hand-written, or source-generated (Mapperly). No reflection-based
  mapping on the settlement path.
- **The provider's field remapping is configuration, not code.** Because they can remap seven
  request fields and all three response fields, the mapping layer reads field names from config so
  a provider-side change is a values change rather than a deployment.
- `Money` conversion to and from decimal strings happens here, at the wire edge, not in any domain.

## Consequences

- **Positive:** one place to reason about provider failure. "How do we behave when CasinoApiPro is
  down?" has a single answer in a single service.
- **Positive:** the internal model is free to be shaped by our domain rather than by their JSON,
  and their configurable field names cannot reach it.
- **Positive:** the inbound rule converts our worst failure mode — Wallet unreachable — from "lost
  settlements" into "settlements retried successfully a moment later."
- **Cost:** Integration now carries two quite different workloads — fast settlement translation and
  slower, retry-heavy outbound work. Keep them from interfering: no catalogue sync on the
  settlement path, separate resilience pipelines, and if they ever contend, split into two
  deployments of the same service (noted as a risk in the architecture doc).
- **Cost:** an extra mapping layer. That is the price of the boundary, and it is the cheapest part
  of this ADR.
- **Testing requirement:** the wire-level provider stub (ADR 0007) must be able to inject timeouts,
  500s and slow responses, so the breaker and retry policies are actually exercised rather than
  assumed. A test must assert that a Wallet outage produces a 5xx and not a mapped business error.

## Alternatives considered

- **Resilience in each calling service.** Rejected: duplicates policy, and leaks provider failure
  semantics into services that must not know the provider exists.
- **A service mesh (Istio/Linkerd) for retries and breaking.** Genuinely capable, and it handles
  the internal hops well. Rejected for the provider boundary specifically: the policies here are
  endpoint-aware in ways a mesh is not — `POST /v1/sessions` must be treated differently from
  `GET /v1/games` because one is not idempotent, and "serve a stale catalogue when the breaker is
  open" is application behaviour, not transport behaviour. A mesh may still be added later for the
  internal hops.
- **A circuit breaker on the inbound settlement path**, for symmetry. Rejected on the analysis
  above: it risks returning a terminal error code for a transient fault, which loses settlements
  the provider would otherwise have retried successfully.
- **Sharing provider DTOs as internal contracts** to skip a mapping layer. Rejected: it couples
  every downstream service to a wire format the provider can reconfigure, and it is exactly the
  coupling ADR 0006's anti-corruption rule exists to prevent.
- **AutoMapper for the mapping layer.** Rejected: reflection-based mapping on a latency-sensitive
  path, with mapping errors surfacing at runtime rather than compile time. Explicit or
  source-generated mapping instead.
