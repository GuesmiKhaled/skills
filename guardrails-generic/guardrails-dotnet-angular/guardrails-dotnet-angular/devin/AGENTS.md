# AGENTS.md — {{MODULE_CIBLE}} (banking / online payment)

> Fill the TODO items, then commit this file at the repo root. Full rules live in `docs/guardrails/` (copy files 01–06 there).

## Stack
- Backend: C# 14, .NET 10, ASP.NET Core, EF Core 10. Databases: SQL Server, Oracle, PostgreSQL (code must not assume one engine).
- Frontend: Angular 20+ (standalone, signals, built-in control flow). Mobile (Flutter) is out of scope: do not touch it.
- Compliance: TODO (e.g. PCI-DSS, GDPR). Secrets store: TODO.

## Commands (TODO: adjust)
- Build: `dotnet build -warnaserror`
- Test: `dotnet test` (includes architecture tests)
- Mutation: `dotnet stryker`
- Security: `dotnet list package --vulnerable --include-transitive`, `gitleaks detect`
- Frontend: `npm ci && npm run lint && npm test && npm audit --audit-level=high`

## Before every PR
1. Run the commands above; everything must pass.
2. Read the matching file in `docs/guardrails/` for the area you changed (map at the bottom).
3. In the PR description list any guardrail you could not satisfy and why. Never silence an analyzer or architecture test to make a build pass.

## Non-negotiable rules (🔴 BLOCKERS)
Money: use `decimal` only; never `double`/`float`; never `(decimal)double`; always explicit `MidpointRounding`; scale from the currency (TND = 3 decimals); Angular never computes money with `number` (amounts are strings or minor units). [B-01]
Idempotency: every effectful POST requires `Idempotency-Key`; unique index on (key, scope); retries reuse the same key. [B-02]
Resilience: external calls only via `IHttpClientFactory` with timeout + circuit breaker + retry with jitter; never `new HttpClient()`; no retry of unsafe methods without the key. [B-03]
Async: forward `CancellationToken`; never swallow `OperationCanceledException`; no `.Result`/`.Wait()`; no `async void`. [B-04]
Messaging: `Acks.All` + idempotent producer; manual commit AFTER the effect; DLQ for poison messages; transactional outbox; TLS only. [B-05]
Sagas: no `TransactionScope`/MSDTC across services; explicit state enum; reconciliation job. [B-06]
Health: `/health/live` + `/health/ready`, K8s probes, graceful shutdown. [B-07]
Secrets: no literal secrets anywhere (C#, JSON, YAML, Dockerfile, `environment.ts`). [S-04]
Sensitive data: never log or store PAN/IBAN/passwords in clear; beware `record.ToString()`; encrypt columns with AES-256-GCM (random 12-byte nonce, key from secret store). [S-02, S-08]
Callbacks: HMAC-SHA256 verified with `CryptographicOperations.FixedTimeEquals`, then timestamp (±5 min) and atomic nonce check. [S-03]
SQL: no concatenation/interpolation into `FromSqlRaw`/`ExecuteSqlRaw`/Dapper; whitelist for ORDER BY. [S-05]
Validation: `<Nullable>enable</Nullable>`, validate every effectful DTO, no `null!`, no `return null` under a non-nullable signature. [S-06]
Rate limiting: every effectful endpoint has a rate-limit policy returning 429 + `Retry-After`. [S-07]
Errors: global `IExceptionHandler` returning RFC 9457 ProblemDetails; no stack in responses; no empty `catch`. [A-03]
Architecture: Domain project has no framework reference; no business rule in controllers/consumers/repositories. [A-01]
API: no field removed/renamed/retyped without versioning (`/v2`) and a deprecation period. [A-02]
Tracing: `X-Correlation-Id` in log scope, propagated to HTTP calls and messages. [A-04]
Persistence: no N+1, no unbounded `ToListAsync`, keyset pagination, atomic `ExecuteUpdateAsync` or concurrency token for balances/counters (Oracle needs an explicit version column). [D-01]
Migrations: expand-contract; never rename/drop in one step; NOT NULL needs a default; never edit an applied migration; no `Database.Migrate()` at startup of several replicas. [D-03]
Resources: `using`/`await using` for every disposable; no network call inside a DB transaction. [D-04]
Dependencies: no AGPL/GPL/SSPL; no CVE with CVSS ≥ 9.0; no floating versions. [S-01]
Angular: `@if/@for` with `track`, signals, `inject()`, OnPush; no `bypassSecurityTrust*`, no `innerHTML` on unsanitised data; every API view handles loading/success/empty/error and never shows a stack. [F-01, F-02, F-03]
Quality (🟡): tests assert exact values and boundaries; cyclomatic ≤ 10, cognitive ≤ 15; no magic numbers; constants are PascalCase. [T-01, T-02]

## When asked to review code
Report each violation as:
`[ID] 🔴 BLOCKER | 🟡 WARNING` / Problem / Rule / Fix / Checklist. Separate facts (what the code does) from interpretation (assumed impact). If information is missing (engine, broker, config), say so instead of guessing.

## Where the full rules are
| Area changed | Read |
|---|---|
| Secrets, logging, encryption, dependencies, callbacks, SQL, validation, rate limits | `docs/guardrails/01-security-S.md` |
| Money, idempotency, resilience, async, messaging, saga, health | `docs/guardrails/02-backend-B.md` |
| Angular | `docs/guardrails/03-frontend-F.md` |
| EF Core, analytics DB, migrations, pools | `docs/guardrails/04-persistence-D.md` |
| Project structure, API contracts, errors, tracing | `docs/guardrails/05-architecture-A.md` |
| Tests, complexity | `docs/guardrails/06-quality-T.md` |
