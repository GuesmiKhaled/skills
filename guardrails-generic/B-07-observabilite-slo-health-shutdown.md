---
id: "B-07"
titre: "Observabilité — Métriques, SLO, Health Checks & Graceful Shutdown"
severite: "🔴 BLOCAGE (service sans health / arrêt sans drain) — 🟡 ALERTE"
applicable_si: [backend]
declencheurs:
  - "Tout nouveau service sans health check et métriques dans {{MODULE_CIBLE}}"
  - "Tout manifest K8s sans livenessProbe/readinessProbe"
  - "Tout service sans SLO défini"
  - "Tout déploiement/arrêt sans drain explicite"
version: "1.0"
---

## Risque
Un service sans `readinessProbe` → trafic envoyé à un pod non prêt → erreurs 5xx en prod.
Un arrêt sans drain → une opération in-flight est abandonnée → état incohérent sans trace → incident.
Sans SLO → on ne sait pas quand le service commence à dégrader silencieusement.

## Règles

**Health Checks & Métriques :**
- Tout service → `{{HEALTH_ENDPOINT}}` (liveness + readiness) + `{{METRICS_ENDPOINT}}` (Prometheus).
- K8s manifest : `livenessProbe` + `readinessProbe` configurées.
- Métriques minimales : `{{OPERATION_TYPE}}.success.rate`, `{{OPERATION_TYPE}}.latency.p99`, taux de rejets.

**SLO à définir :**
- Taux de succès {{OPERATION_TYPE}} ≥ {{SUCCESS_RATE_TARGET}} % → alerte si inférieur.
- Latence P99 ≤ {{P99_TARGET}} ms → alerte si dépassé.
- Opérations en état intermédiaire > {{INTERMEDIATE_COUNT}} depuis > {{INTERMEDIATE_TIMEOUT}} → alerte.

**Graceful Shutdown (séquence obligatoire) :**
1. `readiness` → `false` (LB n'envoie plus de trafic).
2. Drain in-flight (borné, ex. {{DRAIN_TIMEOUT}}).
3. Flush : commit des offsets/messages **après** l'effet.
4. Arrêt propre.
- K8s : `lifecycle.preStop` + `terminationGracePeriodSeconds` ≥ drain + flush.
- Tout in-flight non garanti → état intermédiaire + réconciliation (jamais abandonné).

## Anti-pattern
```yaml
# ❌ BLOCAGE — pas de readinessProbe
containers:
  - name: {{MODULE_CIBLE}}
    image: {{MODULE_CIBLE}}:latest
    # Aucun probe → trafic envoyé même si le service n'est pas prêt
```

## Pattern conforme
```yaml
# ✅ Probes K8s + graceful shutdown
containers:
  - name: {{MODULE_CIBLE}}
    livenessProbe:
      httpGet: { path: {{HEALTH_LIVENESS_PATH}}, port: {{PORT}} }
      initialDelaySeconds: 30
      periodSeconds: 10
    readinessProbe:
      httpGet: { path: {{HEALTH_READINESS_PATH}}, port: {{PORT}} }
      initialDelaySeconds: 20
      periodSeconds: 5
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "sleep {{PRESTOP_DELAY}}"]
spec:
  terminationGracePeriodSeconds: {{TERMINATION_GRACE_PERIOD}}
```

```yaml
# ✅ Graceful shutdown applicatif (Spring Boot)
server.shutdown: graceful
spring.lifecycle.timeout-per-shutdown-phase: {{DRAIN_TIMEOUT}}
```

## Checklist CI/CD
- [ ] `kubectl get deploy {{MODULE_CIBLE}} -o yaml | grep -A5 readinessProbe` → présente
- [ ] `curl {{HEALTH_ENDPOINT}}` → `{"status":"UP"}` avec composants
- [ ] Test : `kill -TERM` pendant une opération in-flight → 0 opération abandonnée, état cohérent
- [ ] Alerte : `{{OPERATION_TYPE}}_success_rate < {{SUCCESS_RATE_TARGET}}` → notification équipe
- [ ] Alerte : `{{OPERATION_TYPE}}_intermediate_count{age>{{INTERMEDIATE_TIMEOUT}}} > {{INTERMEDIATE_COUNT}}`
