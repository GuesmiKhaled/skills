---
id: "B-03"
titre: "Résilience — Circuit Breaker + Retry + Fallback"
severite: "🔴 BLOCAGE (appel externe sans timeout/CB) — 🟡 ALERTE (retry naïf)"
applicable_si: [backend]
declencheurs:
  - "Tout appel synchrone à un service externe depuis {{MODULE_CIBLE}} : {{EXTERNAL_SERVICES}}"
  - "Toute boucle de retry ou annotation de retry dans le code"
  - "Toute opération à effet réessayée sans clé d'idempotence"
version: "1.0"
---

## Risque
Sans timeout : un service externe lent = threads bloqués → pool épuisé → OOM → cascade sur tous les endpoints de {{MODULE_CIBLE}}.
Sans jitter : N instances qui re-tentent en même temps = retry storm → panne prolongée.
Retry à effet sans même `{{IDEMPOTENCY_HEADER}}` = double effet garanti.

## Règles

**Circuit Breaker ({{RESILIENCE_LIB}}) — par dépendance :**
- Timeout explicite : connect ≤ {{CONNECT_TIMEOUT}}, read ≤ budget SLO restant.
- Fenêtre glissante {{CB_WINDOW}} appels, seuil {{CB_THRESHOLD}} %, délai fermeture {{CB_DELAY}}, half-open {{CB_HALF_OPEN}} tentatives.
- Fallback à effet → persister état intermédiaire + événement de réconciliation — jamais échec silencieux.
- Fallback non-critique → valeur dégradée (cache/vide) + log d'alerte.

**Retry — politique unique :**
- Base {{RETRY_BASE}}, facteur {{RETRY_FACTOR}}, **jitter plein**, `maxAttempts = {{RETRY_MAX}}`, cap {{RETRY_CAP}}.
- Retry à effet → **même** clé d'idempotence originale — jamais nouvelle clé (fiche B-02).
- Pas de retry si le circuit breaker est ouvert.

## Anti-pattern
```java
// ❌ BLOCAGE — appel externe sans timeout ni CB
{{SERVICE_RESPONSE}} result = {{EXTERNAL_CLIENT}}.call(request); // peut attendre ∞

// ❌ BLOCAGE — retry naïf
for (int i = 0; i < 10; i++) {
    Thread.sleep(1000); // fixe → storm ; pas d'idempotence → double effet
    {{EXTERNAL_CLIENT}}.call(request);
}
```

## Pattern conforme
```java
// ✅ Resilience4j : CB + timeout + retry
@CircuitBreaker(name = "{{CB_NAME}}", fallbackMethod = "handleFallback")
@Retry(name = "{{RETRY_NAME}}")
@TimeLimiter(name = "{{TIMEOUT_NAME}}")
public CompletableFuture<{{SERVICE_RESPONSE}}> call({{REQUEST}} req) {
    return CompletableFuture.supplyAsync(() -> {{EXTERNAL_CLIENT}}.execute(req));
}

// ✅ Fallback — état intermédiaire + réconciliation
CompletableFuture<{{SERVICE_RESPONSE}}> handleFallback({{REQUEST}} req, Exception ex) {
    repository.updateStatus(req.id(), PENDING);
    eventPublisher.publish(new {{RETRY_EVENT}}(req.id(), req.idempotencyKey()));
    return CompletableFuture.completedFuture({{SERVICE_RESPONSE}}.pending(req.id()));
}
```

```yaml
resilience4j:
  circuitbreaker.instances.{{CB_NAME}}:
    slidingWindowSize: {{CB_WINDOW}}
    failureRateThreshold: {{CB_THRESHOLD}}
    waitDurationInOpenState: {{CB_DELAY}}
    permittedNumberOfCallsInHalfOpenState: {{CB_HALF_OPEN}}
  retry.instances.{{RETRY_NAME}}:
    maxAttempts: {{RETRY_MAX}}
    waitDuration: {{RETRY_BASE}}
    enableExponentialBackoff: true
    exponentialBackoffMultiplier: {{RETRY_FACTOR}}
    enableRandomizedWait: true
    randomizedWaitFactor: 1.0
  timelimiter.instances.{{TIMEOUT_NAME}}:
    timeoutDuration: {{READ_TIMEOUT}}
```

## Checklist CI/CD
- [ ] Tout appel `{{EXTERNAL_SERVICES}}` annoté `@CircuitBreaker` + `@TimeLimiter`
- [ ] Test : coupure {{EXTERNAL_SERVICE}} {{CB_DELAY}} → CB ouvert < 1 s, 0 thread bloqué > timeout, état PENDING persisté
- [ ] Test : retry {{RETRY_MAX}}× échec → exactement {{RETRY_MAX}} tentatives, puis état intermédiaire
- [ ] Test : reprise post-panne → tentatives décorrélées (jitter), pas de pic synchronisé
- [ ] Test : double appel (même clé) → 1 seul effet
