---
id: "S-04"
titre: "Zéro secret en dur — code, config, Dockerfile, CI"
severite: "🔴 BLOCAGE"
applicable_si: [backend, frontend, mobile]
declencheurs:
  - "Littéral type password/secret/private-key/DSN/API-key dans le code, YAML, config, Dockerfile, CI"
  - "Connexion à {{DATA_STORE}} (BDD, cache, queue) avec identifiants en clair"
  - "Clé API, token OAuth, clé de chiffrement en dur dans {{MODULE_CIBLE}}"
version: "1.0"
---

## Risque
Un secret commité dans git = exposé pour toujours (historique git non effaçable).
Un DSN en clair exposé via un endpoint de diagnostics = accès à {{DATA_STORE}} en production.

## Règles

- Tout secret → variable d'environnement ou gestionnaire de secrets : `${{{SECRET_ENV_VAR}}}`.
- Jamais de valeur littérale pour : `password`, `secret`, `key`, `token`, `dsn`, `apiKey`, `privateKey`.
- Fichiers `.env` → hors git (`.gitignore`), fournis via {{SECRET_STORE}}.
- Certificats, keystores → volumes montés, jamais buildés dans l'image.
- CI/CD : variables de pipeline injectées, jamais dans les logs de build.

## Anti-pattern
```yaml
# ❌ BLOCAGE — DSN en clair dans la config
datasource:
  url: jdbc:postgresql://{{DB_HOST}}:5432/{{DB_NAME}}
  password: SuperSecret123  # commité, visible dans git
```

```java
// ❌ BLOCAGE — clé API en dur
private static final String API_KEY = "sk-prod-abcd1234";
```

## Pattern conforme
```yaml
# ✅ Référence à une variable d'environnement
datasource:
  url: ${{{DB_URL_ENV_VAR}}}
  password: ${{{DB_PASSWORD_ENV_VAR}}}  # injecté depuis {{SECRET_STORE}}
```

```yaml
# ✅ Gestionnaire de secrets (exemple Kubernetes)
apiVersion: v1
kind: Secret
metadata:
  name: {{MODULE_CIBLE}}-secrets
type: Opaque
stringData:
  {{DB_PASSWORD_ENV_VAR}}: "{{ vault_lookup('path/to/secret') }}"
```

## Checklist CI/CD
- [ ] `git-secrets --scan` ou `trufflehog` dans le pipeline → 0 secret détecté
- [ ] `grep -r "password\s*=\s*['\"][^$\{]" src/` → 0 valeur littérale
- [ ] Secret `{{MODULE_CIBLE}}-secrets` existe dans {{SECRET_STORE}} pour chaque environnement
- [ ] Aucun `ARG` contenant un secret dans le Dockerfile
