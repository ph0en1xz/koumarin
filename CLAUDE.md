# Koumarin

Read **`TECHSTACK.md` first** — it is the authoritative, fast-load stack reference. This file
carries product context, decisions, and conventions.

**This repo lives in WSL at `/home/mariosl/repos/koumarin`** (Ubuntu 22.04.3, user `mariosl`).
Linux is the platform end to end — dev, build, images, and deployment. Run every command inside
WSL, never from Windows, and never move this repo under `/mnt/c`.

Worked on from the agentic home base at `C:\Users\mario\Claude\Projects\Agentic AI project`
(that workspace is on Windows — the one intentional cross-boundary path). Its `CLAUDE.md` rules
— approval gate, secrets, least privilege, tagging, tests — apply here in full.

---

## What Koumarin is

A **play-money** online casino. Games come from **CasinoApiPro**; Koumarin is the *operator*.

The split that defines the whole system:

> **They run the games. We own the wallet, the ledger, and the player.**

Traffic flows both ways, and the second direction is the real work:
1. **Koumarin → CasinoApiPro:** exchange credentials for a token, list games, create a session,
   get a signed `launch_url`, embed it in an `<iframe>`.
2. **CasinoApiPro → Koumarin:** every bet and every win calls *our* wallet API. They never hold
   a balance. Our response body carries the new balance.

So the backend is, in essence, **a wallet and ledger service** with a provider gateway in front
of it and a reporting consumer behind it.

No real money, no KYC, no payment rails, no gaming licence. See the locked constraints in
`TECHSTACK.md` — lifting any of them is a licensing decision, not an engineering one.

## Why the architecture is "too big" on purpose

Microservices + RabbitMQ + EKS is heavier than a play-money demo needs. There is no independent
scaling axis, no second team, and no load. That was raised and accepted on 2026-10-08: the
architecture is itself part of the deliverable — a .NET / Linux / Kubernetes / messaging
portfolio piece.

What was *not* accepted is splitting the ledger. The one place this design stays conservative is
money: the wallet's debit/credit path is a single Postgres transaction in a single service, and
it never makes a network call to answer. Distributed everywhere else, strongly consistent there.

## The three invariants

Everything else is negotiable. These are not:

1. **Idempotency on `transactionId`.** Unique index; check *inside* the transaction; a replay
   returns the balance returned the first time and sets `duplicate: true`. It does not move money
   twice. A 6-second timeout on their side is ambiguous — the money may have moved and the
   response been lost — which is why this is load-bearing rather than defensive.
2. **Atomicity.** The money movement, the transaction id, and the outbox row commit together or
   not at all.
3. **`decimal`, never float.** Money is `decimal` in C# and a decimal string (`"10.50"`) on the
   wire. Any binary float on a monetary path is a defect, not a style choice.

## Current phase

**Pre-architecture.** Repo scaffolded 2026-10-08; no code yet.

Two prerequisites before anything builds, both **inside WSL**:
- **.NET 10 SDK** — not installed (the Windows-side 8.0.202 does not count, and .NET 8 is EOL
  2026-11-10 regardless).
- **helm** — not installed, if the deploy path turns out to be Helm.

Next: the architecture session (`solution-architect` skill) on the wallet/ledger — the data model,
the outbox, the HMAC verification seam, and the service boundaries. That produces
`documentation/architecture/koumarin/architecture.md` in the home base plus ADRs in `docs/adr/`
here, **before** any `.csproj` exists. Project layout is an output of that session, not an input.

A fast first win is available and worth taking early: in sandbox you can create a test player in
CasinoApiPro's dashboard and launch a real game **before** writing any wallet code. Prove the
loop end to end, then build the wallet properly behind it.

## Conventions

- **Git:** conventional commits (`feat:`, `fix:`, `chore:`, `docs:`). Branch per feature; never
  commit directly to `main`. **Never commit or push without explicit approval.**
- **ADRs:** `docs/adr/` here, numbered, linked from the home-base architecture doc.
- **Runbooks:** in the home base at `documentation/runbooks/koumarin/<ticket>/runbook.md`.
- **Tests:** every code change. Testcontainers against real Postgres on the WSL Docker daemon —
  never the EF Core InMemory provider on anything with ledger or idempotency semantics.
- **Secrets:** never in the repo. SSM/Secrets Manager → External Secrets. `.env` is local-only.
- **Shell:** POSIX only. No PowerShell scripts in this repo.

## Known traps

- **HMAC verification:** read the **raw request body bytes** before model binding. If ASP.NET
  Core deserializes and re-serializes, the signature will not match and the failure looks like a
  wrong secret.
- **Token caching:** CasinoApiPro tokens live 3600s. Cache in Redis; minting one per request
  burns rate limit for nothing.
- **Sessions vs rounds:** stakes and open rounds bind to the **player**, not the session. Closing
  a session moves no money and does not settle in-play rounds; players resume unfinished games in
  new sessions. Do not model rounds as session-scoped.
- **Launch URLs** expire in minutes and are scoped to one operator/player/game. Mint fresh on
  every open; never cache or reuse one.
- **WSL shell quoting:** running a command through `wsl.exe -- bash -c "…"` from Git Bash lets the
  *outer* shell expand `$(…)` and `$VAR` first, so output can silently describe the Windows side
  instead of WSL. Escape as `\$(…)` or use single quotes. This has produced misleading output before.
- **Line endings:** LF enforced via `.gitattributes`. The repo is edited from a Windows IDE onto
  the Linux filesystem, so a CRLF leak would otherwise break shell scripts at runtime.
