# Guardrails Generic — v1.0

> **29 fiches** · Dossier générique avec variables `{{VAR}}` à substituer selon votre contexte · Format court 40-60 lignes

---

## Pourquoi ce corpus existe

Ce dossier est la **version générique recomposée** (`06-guardrails-generic-v2`) du corpus Spark V2 (`06-guardrails-v2`).
Les exemples spécifiques à Spark V2 sont remplacés par des variables `{{VAR}}` à adapter.

Usage :
- Remplacez `{{MODULE_CIBLE}}` par le nom de votre service ou module.
- Remplacez `{{SENSITIVE_DATA_TYPE}}` par le type de données sensibles de votre domaine.
- Les fiches `applicable_si` vous permettent de charger uniquement les fiches pertinentes.

---

## Variables globales à substituer

### Identité du projet

| Variable | Description | Exemple Spark V2 |
|----------|-------------|-----------------|
| `{{MODULE_CIBLE}}` | Nom du service/module audité | `cp-backend`, `cp-frontend` |
| `{{DOMAIN_DESCRIPTION}}` | Description du domaine métier | `systèmes de paiement monétiques` |
| `{{TECH_STACK}}` | Stack technique | `Java 21, Spring Boot 3, Angular 17, PostgreSQL, Kafka` |
| `{{COMPLIANCE_STANDARD}}` | Norme de conformité applicable | `PCI-DSS`, `RGPD`, `HIPAA` |
| `{{BASE_PACKAGE}}` | Package racine du projet | `tech.mssolutions`, `com.mycompany` |

### Données & sécurité

| Variable | Description | Exemple Spark V2 |
|----------|-------------|-----------------|
| `{{SENSITIVE_DATA_TYPE}}` | Type de donnée sensible | `PAN, CVV, IBAN, SSN` |
| `{{SENSITIVE_FIELD}}` | Nom de colonne sensible | `instrument_ref`, `pan_encrypted` |
| `{{SENSITIVE_KEYWORD}}` | Mot-clé pour grep dans les logs | `pan`, `cvv`, `iban` |
| `{{SECRET_STORE}}` | Gestionnaire de secrets | `HashiCorp Vault`, `AWS Secrets Manager` |
| `{{SECRET_ENV_VAR}}` | Nom de la variable d'environnement du secret | `SPARK_DB_PASSWORD`, `APP_API_KEY` |
| `{{ENCRYPTION_CONVERTER}}` | Classe de chiffrement JPA | `AesGcmEncryptor` |
| `{{ENCRYPTION_KEY_ENV_VAR}}` | Variable d'env pour la clé de chiffrement | `SPARK_COLUMN_ENCRYPTION_KEY` |

### BDD & persistance

| Variable | Description | Exemple Spark V2 |
|----------|-------------|-----------------|
| `{{TABLE}}` / `{{ENTITY}}` | Table/entité principale | `payment`, `Transaction` |
| `{{DATA_STORE}}` | Type de stockage principal | `PostgreSQL`, `MySQL`, `MongoDB` |
| `{{ANALYTICS_DB}}` | BDD analytique utilisée | `ClickHouse`, `BigQuery`, `Redshift` |
| `{{TRANSACTIONAL_DB}}` | BDD transactionnelle | `PostgreSQL`, `MySQL` |
| `{{MIGRATION_TOOL}}` | Outil de migration | `Flyway`, `Liquibase`, `Alembic` |
| `{{CONNECTION_POOL_LIB}}` | Librairie de pool de connexions | `HikariCP`, `c3p0`, `DBCP` |
| `{{CONNECTION_TIMEOUT}}` | Timeout de connexion BDD | `500` (ms) |

### API & communication

| Variable | Description | Exemple Spark V2 |
|----------|-------------|-----------------|
| `{{ENDPOINT}}` | Endpoint principal | `payments`, `authorizations` |
| `{{IDEMPOTENCY_HEADER}}` | En-tête d'idempotence | `Idempotency-Key` |
| `{{CORRELATION_HEADER}}` | En-tête de corrélation | `X-Correlation-Id` |
| `{{SIGNATURE_FILTER}}` | Filtre de vérification de signature | `WebhookSignatureFilter` |
| `{{ERROR_FORMAT}}` | Format de réponse d'erreur | `RFC 7807 ProblemDetail` |
| `{{DEPRECATION_PERIOD}}` | Durée minimale de dépréciation | `2 sprints`, `1 mois` |

### Messaging

| Variable | Description | Exemple Spark V2 |
|----------|-------------|-----------------|
| `{{MESSAGE_BROKER}}` | Broker de messages | `Kafka`, `RabbitMQ`, `SQS` |
| `{{ACKS_CONFIG}}` | Config d'acquittement producer | `acks` (Kafka), `durability` (RabbitMQ) |
| `{{DLQ_TOPIC}}` | Nom du topic Dead Letter Queue | `spark.payment.dlq` |
| `{{SECURITY_PROTOCOL}}` | Protocole de sécurité messaging | `SSL`, `SASL_SSL`, `TLS` |

### Résilience

| Variable | Description | Exemple Spark V2 |
|----------|-------------|-----------------|
| `{{RESILIENCE_LIB}}` | Librairie de résilience | `Resilience4j`, `Hystrix`, `Polly` |
| `{{CB_NAME}}` | Nom du Circuit Breaker | `switch-cb`, `payment-cb` |
| `{{CB_WINDOW}}` | Taille de la fenêtre glissante CB | `20` |
| `{{CB_THRESHOLD}}` | Seuil d'échec CB (%) | `50` |
| `{{CB_DELAY}}` | Délai d'ouverture CB | `30s` |
| `{{CB_HALF_OPEN}}` | Tentatives en half-open | `3` |
| `{{RETRY_MAX}}` | Nombre max de tentatives | `6` |
| `{{RETRY_BASE}}` | Délai de base du retry | `500ms` |
| `{{RETRY_FACTOR}}` | Facteur exponentiel retry | `2` |
| `{{EXTERNAL_SERVICES}}` | Services externes appelés | `switch ISO 8583`, `HSM`, `PSP` |

### Framework & architecture

| Variable | Description | Exemple Spark V2 |
|----------|-------------|-----------------|
| `{{FRAMEWORK_PACKAGES}}` | Packages framework interdits dans le domaine | `..springframework..`, `..jakarta..` |
| `{{INTEGRATION_TEST_ANNOTATION}}` | Annotation de test d'intégration | `@SpringBootTest`, `@QuarkusTest` |
| `{{FRONTEND_FRAMEWORK}}` | Framework frontend utilisé | `Angular 17`, `React 18`, `Vue 3` |
| `{{DEPRECATED_PATTERNS}}` | Patterns dépréciés à éviter | `NgModule`, `*ngIf`, `*ngFor` |

### SLO & observabilité

| Variable | Description | Exemple Spark V2 |
|----------|-------------|-----------------|
| `{{SUCCESS_RATE_TARGET}}` | Taux de succès cible (%) | `99.5` |
| `{{P99_TARGET}}` | Latence P99 cible (ms) | `500` |
| `{{HEALTH_ENDPOINT}}` | Endpoint de health check | `/actuator/health` |
| `{{METRICS_ENDPOINT}}` | Endpoint de métriques | `/actuator/metrics` |

### Qualité

| Variable | Description | Exemple Spark V2 |
|----------|-------------|-----------------|
| `{{MUTATION_THRESHOLD}}` | Seuil mutation testing (%) | `70` |
| `{{MUTATION_TOOL}}` | Outil de mutation testing | `PIT`, `Stryker`, `mutmut` |
| `{{CRITICAL_CLASSES}}` | Classes critiques à tester | `RefundPolicy`, `FeeCalculator` |
| `{{CYCLO_THRESHOLD}}` | Seuil complexité cyclomatique | `10` |
| `{{COGNITIVE_THRESHOLD}}` | Seuil complexité cognitive | `15` |
| `{{MAX_PR_LINES}}` | Taille max d'une PR (hors tests) | `400` |
| `{{DOMAIN}}` | Domaine métier pour le scope de commit | `payment`, `auth`, `settlement` |

> **Note** : les variables du tableau ci-dessus sont des **variables de configuration obligatoires** à substituer avant d'utiliser le corpus.
> Les jetons `{{...}}` présents dans les **exemples de code** des fiches (ex. `{{COMPREHENSION_TIME}}`, `{{DATE}}`, `{{ITEM}}`) sont des **slots d'illustration** à remplacer par vos valeurs de domaine — ils ne font pas partie des variables de configuration.

---

## Format des fiches (gabarit v2.0)

```markdown
## Risque          — 2 lignes max, impact business concret
## Règles          — bullets courts, règles actionnables
## Anti-pattern    — code ❌ avec commentaire
## Pattern conforme — code ✅ avec commentaire
## Checklist CI/CD — commandes exécutables ou tests verts
```

---

## Matrice des fiches — 29 fiches, 6 blocs

### Bloc S — Sécurité (8 fiches)

| ID   | Titre | Absorbe | applicable_si |
|------|-------|---------|---------------|
| S-01 | SBOM, licences & CVE scan | 01 | all |
| S-02 | Zéro donnée sensible — PII, logs, chiffrement | 02 + 05 | backend, frontend |
| S-03 | Callbacks — HMAC + anti-rejeu | 06 + 07 | backend |
| S-04 | Zéro secret en dur | 04 | all |
| S-05 | SQL Injection | 42 | backend |
| S-06 | Validation DTO + null safety | 40 + 41 | backend |
| S-07 | Rate Limiting & Throttling | 44 | backend |
| S-08 | Chiffrement au repos | 45 | backend |

### Bloc B — Backend / Résilience (7 fiches)

| ID   | Titre | Absorbe | applicable_si |
|------|-------|---------|---------------|
| B-01 | Money Safety — BigDecimal | 03 | backend |
| B-02 | Idempotence des opérations à effet | 08 | backend |
| B-03 | Résilience — CB + Retry + Fallback | 12 + 13 | backend |
| B-04 | Threads — Virtual Threads + Concurrence | 09 + 43 | backend |
| B-05 | Messaging — Delivery, DLQ, Events, chiffrement | 10 + 29 | backend |
| B-06 | Saga — transactions distribuées | 14 | backend |
| B-07 | Observabilité — SLO, Health, Shutdown | 35 + 15 | backend |

### Bloc F — Frontend / Mobile (4 fiches)

| ID   | Titre | Absorbe | applicable_si |
|------|-------|---------|---------------|
| F-01 | {{FRONTEND_FRAMEWORK}} — Composants modernes | 16 | frontend |
| F-02 | XSS — Sanitisation du contenu dynamique | 17 | frontend |
| F-03 | Mobile — Secure Storage + Sync offline | 18 + 20 | mobile |
| F-04 | Error Boundaries — format d'erreur structuré | 19 | frontend |

### Bloc D — Persistance / BDD (4 fiches)

| ID   | Titre | Absorbe | applicable_si |
|------|-------|---------|---------------|
| D-01 | BDD — Anti N+1, keyset, locking | 21 + 23 | backend |
| D-02 | {{ANALYTICS_DB}} — requêtes analytiques | 22 | backend |
| D-03 | Migration zero-downtime | 24 | backend |
| D-04 | Pool de connexions + resource leak | 25 | backend |

### Bloc A — Architecture (4 fiches)

| ID   | Titre | Absorbe | applicable_si |
|------|-------|---------|---------------|
| A-01 | Clean Architecture — isolation, SRP, naming | 26 + 28 + 36 | backend |
| A-02 | Backward Compatibility API | 30 | backend, frontend |
| A-03 | Exception Handling — format structuré | 31 | backend |
| A-04 | Tracing distribué — correlationId | 34 | backend |

### Bloc T — Qualité / Craftsmanship (3 fiches)

| ID   | Titre | Absorbe | applicable_si |
|------|-------|---------|---------------|
| T-01 | TDD + ArchUnit + Mutation Testing | 32 + 27 + 33 | backend |
| T-02 | Complexité & Lisibilité — constantes | 37 + 38 | backend, frontend |
| *(T-03)* | *(Conventional Commits — 5 règles, voir ci-dessous)* | 39 | — |

---

## Conventional Commits — 5 règles (ex-fiche 39)

1. Format : `type(scope): message` — ex. `feat({{DOMAIN}}): add idempotency check`
2. Types autorisés : `feat`, `fix`, `refactor`, `test`, `chore`, `docs`, `perf`, `ci`
3. PR ≤ `{{MAX_PR_LINES}}` lignes (hors tests)
4. Message au présent, impératif : `add`, `fix`, `remove`
5. Breaking change → `!` dans le type : `feat(api)!: remove /v1/{{ENDPOINT}}`

---

## Chargement dynamique par profil

```yaml
applicable_si:
  backend:  [S-01..S-08, B-01..B-07, D-01..D-04, A-01..A-04, T-01..T-02]
  frontend: [S-01, S-02, S-04, F-01..F-04, A-02, T-02]
  mobile:   [S-01, S-04, F-03]
```

---

*Version corpus : v1.0 · Générique · 29 fiches actives*
