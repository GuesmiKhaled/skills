# Guardrails Generic — v1.0

> **29 cards** · Generic folder with `{{VAR}}` variables to substitute per your context · Short 40-60 line format
> Adapted to **.NET 10 / C# 14 / ASP.NET Core / EF Core 10 / Angular 20+**

---

## Why this corpus exists

This folder is the **reworked generic version** (`06-guardrails-generic-v2`) of the Spark V2 corpus (`06-guardrails-v2`).
The Spark V2-specific examples are replaced by `{{VAR}}` variables to adapt.

Usage:
- Replace `{{MODULE_CIBLE}}` with the name of your service or module.
- Replace `{{SENSITIVE_DATA_TYPE}}` with the sensitive data type of your domain.
- The `applies_to` field of each card lets you load only the relevant cards.

---

## Global variables to substitute

### Project identity

| Variable | Description | Example Spark V2 |
|----------|-------------|-----------------|
| `{{MODULE_CIBLE}}` | Name of the service/module audited | `cp-backend`, `cp-frontend` |
| `{{DOMAIN_DESCRIPTION}}` | Description of the business domain | `monetary payment systems` |
| `{{TECH_STACK}}` | Technical stack | `C# 14 / .NET 10, ASP.NET Core, EF Core 10, Angular 20+, SQL Server, Kafka` |
| `{{COMPLIANCE_STANDARD}}` | Applicable compliance standard | `PCI-DSS`, `GDPR`, `HIPAA` |
| `{{BASE_NAMESPACE}}` | Root namespace of the project | `MSSolutions`, `MyCompany` |

### Data & security

| Variable | Description | Example Spark V2 |
|----------|-------------|-----------------|
| `{{SENSITIVE_DATA_TYPE}}` | Sensitive data type | `PAN, CVV, IBAN, SSN` |
| `{{SENSITIVE_FIELD}}` | Name of a sensitive column | `instrument_ref`, `pan_encrypted` |
| `{{SENSITIVE_KEYWORD}}` | Keyword to grep in logs | `pan`, `cvv`, `iban` |
| `{{SECRET_STORE}}` | Secret manager | `Azure Key Vault`, `HashiCorp Vault`, `AWS Secrets Manager` |
| `{{SECRET_ENV_VAR}}` | Name of the secret's environment variable | `SPARK_DB_PASSWORD`, `APP_API_KEY` |
| `{{ENCRYPTION_CONVERTER}}` | EF Core encryption converter class | `AesGcmValueConverter` |
| `{{ENCRYPTION_KEY_ENV_VAR}}` | Environment variable for the encryption key | `SPARK_COLUMN_ENCRYPTION_KEY` |

### DB & persistence

| Variable | Description | Example Spark V2 |
|----------|-------------|-----------------|
| `{{TABLE}}` / `{{ENTITY}}` | Main table/entity | `payment`, `Transaction` |
| `{{DATA_STORE}}` | Main storage type | `SQL Server`, `Oracle`, `PostgreSQL` |
| `{{ANALYTICS_DB}}` | Analytical DB used | `ClickHouse`, `BigQuery`, `Redshift` |
| `{{TRANSACTIONAL_DB}}` | Transactional DB | `SQL Server`, `PostgreSQL` |
| `{{MIGRATION_TOOL}}` | Migration tool | `EF Core migrations`, `Flyway`, `Liquibase` |
| `{{CONNECTION_POOL_LIB}}` | Connection pool library | `Microsoft.Data.SqlClient` (pooling), `EF Core pooling` |
| `{{CONNECTION_TIMEOUT}}` | DB connection timeout | `500` (ms) |

### API & communication

| Variable | Description | Example Spark V2 |
|----------|-------------|-----------------|
| `{{ENDPOINT}}` | Main endpoint | `payments`, `authorizations` |
| `{{IDEMPOTENCY_HEADER}}` | Idempotency header | `Idempotency-Key` |
| `{{CORRELATION_HEADER}}` | Correlation header | `X-Correlation-Id` |
| `{{SIGNATURE_FILTER}}` | Signature verification middleware | `WebhookSignatureMiddleware` |
| `{{ERROR_FORMAT}}` | Error response format | `RFC 9457 ProblemDetails` |
| `{{DEPRECATION_PERIOD}}` | Minimum deprecation duration | `2 sprints`, `1 month` |

### Messaging

| Variable | Description | Example Spark V2 |
|----------|-------------|-----------------|
| `{{MESSAGE_BROKER}}` | Message broker | `Kafka`, `RabbitMQ`, `Azure Service Bus` |
| `{{ACKS_CONFIG}}` | Producer acknowledgment setting | `acks` (Kafka), `delivery mode` (RabbitMQ) |
| `{{DLQ_TOPIC}}` | Dead Letter Queue topic name | `spark.payment.dlq` |
| `{{SECURITY_PROTOCOL}}` | Messaging security protocol | `SASL_SSL`, `SaslSsl`, `TLS` |

### Resilience

| Variable | Description | Example Spark V2 |
|----------|-------------|-----------------|
| `{{RESILIENCE_LIB}}` | Resilience library | `Polly`, `Microsoft.Extensions.Http.Resilience` |
| `{{CB_NAME}}` | Circuit Breaker name | `switch-cb`, `payment-cb` |
| `{{CB_WINDOW}}` | CB sliding window size | `20` |
| `{{CB_THRESHOLD}}` | CB failure threshold (%) | `50` |
| `{{CB_DELAY}}` | CB open duration | `30s` |
| `{{CB_HALF_OPEN}}` | Attempts in half-open | `3` |
| `{{RETRY_MAX}}` | Max retry attempts | `6` |
| `{{RETRY_BASE}}` | Retry base delay | `500ms` |
| `{{RETRY_FACTOR}}` | Retry exponential factor | `2` |
| `{{EXTERNAL_SERVICES}}` | External services called | `ISO 8583 switch`, `HSM`, `PSP` |
### Framework & architecture

| Variable | Description | Example Spark V2 |
|----------|-------------|-----------------|
| `{{FRAMEWORK_PACKAGES}}` | Framework namespaces forbidden in the domain | `Microsoft.AspNetCore`, `Microsoft.EntityFrameworkCore` |
| `{{INTEGRATION_TEST_ANNOTATION}}` | Integration test attribute | `[Collection("Integration")]`, `WebApplicationFactory` |
| `{{FRONTEND_FRAMEWORK}}` | Frontend framework used | `Angular 20+`, `React 18`, `Vue 3` |
| `{{DEPRECATED_PATTERNS}}` | Deprecated patterns to avoid | `NgModule`, `*ngIf`, `*ngFor` |

### SLO & observability

| Variable | Description | Example Spark V2 |
|----------|-------------|-----------------|
| `{{SUCCESS_RATE_TARGET}}` | Target success rate (%) | `99.5` |
| `{{P99_TARGET}}` | Target P99 latency (ms) | `500` |
| `{{HEALTH_ENDPOINT}}` | Health check endpoint | `/health/ready` |
| `{{METRICS_ENDPOINT}}` | Metrics endpoint | `/metrics` (Prometheus) |

### Quality

| Variable | Description | Example Spark V2 |
|----------|-------------|-----------------|
| `{{MUTATION_THRESHOLD}}` | Mutation testing threshold (%) | `70` |
| `{{MUTATION_TOOL}}` | Mutation testing tool | `Stryker.NET`, `Stryker` |
| `{{CRITICAL_CLASSES}}` | Critical classes to test | `RefundPolicy`, `FeeCalculator` |
| `{{CYCLO_THRESHOLD}}` | Cyclomatic complexity threshold | `10` |
| `{{COGNITIVE_THRESHOLD}}` | Cognitive complexity threshold | `15` |
| `{{MAX_PR_LINES}}` | Max PR size (excluding tests) | `400` |
| `{{DOMAIN}}` | Business domain for the commit scope | `payment`, `auth`, `settlement` |

> **Note**: the variables in the tables above are **mandatory configuration variables** to substitute before using the corpus.
> The `{{...}}` tokens present in the **code examples** of the cards (e.g. `{{COMPREHENSION_TIME}}`, `{{DATE}}`, `{{ITEM}}`) are **illustration slots** to replace with your domain values — they are not configuration variables.

---

## Card format (v2.0 template)

```markdown
## Risk           — max 2 lines, concrete business impact
## Rules          — short bullets, actionable rules
## Anti-pattern   — ❌ code with comments
## Compliant pattern — ✅ code with comments
## CI/CD checklist — executable commands or green tests
```

---

## Card matrix — 29 cards, 6 blocks

### Block S — Security (8 cards)

| ID   | Title | Absorbs | applies_to |
|------|-------|---------|---------------|
| S-01 | SBOM, licences & CVE scan | 01 | all |
| S-02 | Zero sensitive data — PII, logs, encryption | 02 + 05 | backend, frontend |
| S-03 | Callbacks — HMAC + anti-replay | 06 + 07 | backend |
| S-04 | Zero hardcoded secrets | 04 | all |
| S-05 | SQL injection | 42 | backend |
| S-06 | DTO validation + null safety | 40 + 41 | backend |
| S-07 | Rate limiting & throttling | 44 | backend |
| S-08 | Encryption at rest | 45 | backend |

### Block B — Backend / Resilience (7 cards)

| ID   | Title | Absorbs | applies_to |
|------|-------|---------|---------------|
| B-01 | Money safety — decimal | 03 | backend |
| B-02 | Idempotency of side-effect operations | 08 | backend |
| B-03 | Resilience — CB + Retry + Fallback | 12 + 13 | backend |
| B-04 | Concurrency — virtual threads + concurrency | 09 + 43 | backend |
| B-05 | Messaging — delivery, DLQ, events, encryption | 10 + 29 | backend |
| B-06 | Saga — distributed transactions | 14 | backend |
| B-07 | Observability — SLO, health, shutdown | 35 + 15 | backend |

### Block F — Frontend / Mobile (4 cards)

| ID   | Title | Absorbs | applies_to |
|------|-------|---------|---------------|
| F-01 | {{FRONTEND_FRAMEWORK}} — modern components | 16 | frontend |
| F-02 | XSS — sanitization of dynamic content | 17 | frontend |
| F-03 | Mobile — secure storage + offline sync | 18 + 20 | mobile |
| F-04 | Error boundaries — structured error format | 19 | frontend |

### Block D — Persistence / DB (4 cards)

| ID   | Title | Absorbs | applies_to |
|------|-------|---------|---------------|
| D-01 | DB — anti N+1, keyset, locking | 21 + 23 | backend |
| D-02 | {{ANALYTICS_DB}} — analytical queries | 22 | backend |
| D-03 | Zero-downtime migrations | 24 | backend |
| D-04 | Connection pool + resource leak | 25 | backend |

### Block A — Architecture (4 cards)

| ID   | Title | Absorbs | applies_to |
|------|-------|---------|---------------|
| A-01 | Clean Architecture — isolation, SRP, naming | 26 + 28 + 36 | backend |
| A-02 | API backward compatibility | 30 | backend, frontend |
| A-03 | Exception handling — structured format | 31 | backend |
| A-04 | Distributed tracing — correlationId | 34 | backend |

### Block T — Quality / Craftsmanship (2 cards)

| ID   | Title | Absorbs | applies_to |
|------|-------|---------|---------------|
| T-01 | TDD + architecture tests + mutation testing | 32 + 27 + 33 | backend |
| T-02 | Complexity & readability — constants | 37 + 38 | backend, frontend |
| *(T-03)* | *(Conventional Commits — 5 rules, see below)* | 39 | — |

---

## Conventional Commits — 5 rules (former card 39)

1. Format: `type(scope): message` — e.g. `feat({{DOMAIN}}): add idempotency check`
2. Allowed types: `feat`, `fix`, `refactor`, `test`, `chore`, `docs`, `perf`, `ci`
3. PR ≤ `{{MAX_PR_LINES}}` lines (excluding tests)
4. Message in the present, imperative: `add`, `fix`, `remove`
5. Breaking change → `!` in the type: `feat(api)!: remove /v1/{{ENDPOINT}}`

---

## Dynamic loading by profile

```yaml
applies_to:
  backend:  [S-01..S-08, B-01..B-07, D-01..D-04, A-01..A-04, T-01..T-02]
  frontend: [S-01, S-02, S-04, F-01..F-04, A-02, T-02]
  mobile:   [S-01, S-04, F-03]
```

---

*Corpus version: v1.0 · Generic · 29 active cards · .NET 10 / Angular 20+ edition*

