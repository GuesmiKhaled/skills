# Devin Knowledge items (create each in Settings & Library → Knowledge)

Pin every item to your repo. Each item = Trigger Description + Content. Keep content short; the details stay in `docs/guardrails/`.
Optional macro (type `!name` in a prompt to force the item): shown in brackets.

---

## 1 [!money] Money and amounts

**Trigger:** When writing or changing code that calculates, stores, compares, rounds or displays money amounts (C# or Angular).
**Content:** Use `decimal` only, never `double`/`float`/`(decimal)double`. Always pass an explicit `MidpointRounding` (default is banker's rounding). Take the scale from the currency (TND 3 decimals, EUR 2, JPY 0). Use a Money value object with currency. In Angular amounts are strings or integer minor units; never do arithmetic on `number`. Details: docs/guardrails/02-backend-B.md (B-01).

## 2 [!endpoint] Effectful API endpoints and callbacks

**Trigger:** When creating or modifying a POST/PUT/DELETE API endpoint, webhook or callback.
**Content:** Require `Idempotency-Key` with a unique index (key, scope). Validate the DTO (nullable enabled, no `null!`). Add a rate-limit policy returning 429 + Retry-After. Return ProblemDetails errors via the global handler. Webhooks: verify HMAC with `CryptographicOperations.FixedTimeEquals`, then timestamp ±5 min, then atomic nonce store. Details: B-02, S-06, S-07, S-03, A-03.

## 3 [!external-calls] External calls, messaging, sagas, async

**Trigger:** When calling external services over HTTP, producing or consuming Kafka/queue messages, writing background services, or coordinating several services.
**Content:** Use IHttpClientFactory with timeout + circuit breaker + retry with jitter (no retry of POST without the idempotency key). Kafka: Acks.All, idempotent producer, manual commit after the effect, DLQ, transactional outbox, TLS. No TransactionScope across services; use a state enum plus reconciliation. Forward CancellationToken; never swallow OperationCanceledException; no .Result/.Wait()/async void. Details: B-03, B-04, B-05, B-06.

## 4 [!ef-core] EF Core queries and concurrent updates

**Trigger:** When writing EF Core queries, repositories, pagination, or updating balances, counters or quotas.
**Content:** No N+1 (use Select projection or Include + AsSplitQuery), AsNoTracking for reads, no lazy-loading proxies, keyset pagination, no unbounded ToListAsync. Balances: atomic ExecuteUpdateAsync with the guard in WHERE, or a concurrency token (SQL Server rowversion, PostgreSQL xmin, Oracle explicit version column) with DbUpdateConcurrencyException mapped to 409. Dispose every connection; no network call inside a transaction. Details: D-01, D-04, S-05.

## 5 [!db-migration] Database migrations

**Trigger:** When creating or editing database migrations or schema changes for SQL Server, Oracle or PostgreSQL.
**Content:** Expand-contract in two releases. EF scaffolds RenameColumn on property rename: rewrite it as AddColumn + backfill. NOT NULL needs a default. Online indexes (PostgreSQL CONCURRENTLY with suppressTransaction, SQL Server ONLINE=ON, Oracle ONLINE). Oracle DDL auto-commits: keep migrations small and re-runnable. Never edit an applied migration; do not call Database.Migrate() at startup of several replicas. Details: D-03.

## 6 [!secrets] Secrets, sensitive data, logs, encryption

**Trigger:** When handling secrets, connection strings, card/IBAN/personal data, logging, or encryption.
**Content:** No literal secrets in code, JSON, YAML, Dockerfile or environment.ts; use env vars / secret store. Never log sensitive objects (record.ToString prints every property); log only whitelisted fields (correlationId, opaque ids, status codes). Encrypt sensitive columns with AES-256-GCM, random 12-byte nonce per value, key version byte, key from the secret store; use a blind index for lookups. Details: S-02, S-04, S-08.

## 7 [!deps] Dependencies

**Trigger:** When adding or updating NuGet or npm packages.
**Content:** Reject AGPL/GPL/SSPL licences and CVSS ≥ 9.0. Check current licence of the package (several .NET libraries went commercial). Use central package management and lock files; no floating versions; prefer versions published at least 7 days ago. Run dotnet list package --vulnerable and npm audit. Details: S-01.

## 8 [!architecture] Structure, API contracts, errors, tracing

**Trigger:** When adding classes or projects, changing API contracts, handling exceptions, or adding logs in the backend.
**Content:** Domain project has no framework references; business rules never in controllers/consumers/repositories. No removing/renaming/retyping API fields without /v2 versioning, Sunset header and deprecation period. Global IExceptionHandler with ProblemDetails, no stack in responses, no empty catch, use `throw;`. Put X-Correlation-Id in the log scope and propagate it. Details: A-01 to A-04.

## 9 [!angular] Angular code

**Trigger:** When writing or changing Angular components, templates, HTTP calls, error states or HTML rendering.
**Content:** Standalone components, signals, input()/output(), inject(), OnPush, @if/@for with track item.id, functional interceptors/guards, lazy routes. Never bypassSecurityTrust* or innerHTML on unsanitised data (use DOMPurify). Every API view has loading/success/empty/error states, maps ProblemDetails errorCode to i18n text, never shows stacks; 4xx final, 5xx retryable with backoff; manual retry of a POST reuses the same idempotency key. Details: F-01 to F-03.

## 10 [!observability] Services, health, deployment

**Trigger:** When adding a service, health checks, Kubernetes manifests, metrics or shutdown logic.
**Content:** /health/live (no dependency checks) and /health/ready; readiness turns false on shutdown; HostOptions.ShutdownTimeout and terminationGracePeriodSeconds aligned with preStop + drain; exec-form ENTRYPOINT; OpenTelemetry metrics for success rate, latency, intermediate-state count. Details: B-07.

## 11 [!tests] Tests and complexity

**Trigger:** When writing or reviewing tests, or when a method becomes complex.
**Content:** Assert exact values and exact boundaries (kills < vs <= mutants); no test without assertion; domain rule tests use no integration infrastructure; keep ArchUnitNET tests (no framework in Domain, no cycles, layering) and Stryker thresholds green; cyclomatic ≤ 10, cognitive ≤ 15; no magic numbers; constants PascalCase. Details: T-01, T-02.
