---
id: "S-07"
titre: "Rate Limiting & Throttling — protection des endpoints à effet"
severite: "🔴 BLOCAGE (endpoint à effet sans rate limiting)"
applicable_si: [backend]
declencheurs:
  - "Tout nouvel endpoint POST/PUT/DELETE à effet dans {{MODULE_CIBLE}}"
  - "Tout endpoint exposé sans quota par identifiant (utilisateur, API key, IP)"
  - "Tout endpoint critique sans protection contre les appels automatisés"
version: "1.0"
---

## Risque
Un endpoint à effet sans rate limiting peut être inondé : {{MAX_CALLS}} appels en {{TIME_WINDOW}} = {{RISK_DESCRIPTION}} (saturation, coût, données incorrectes).
Sans throttling sur {{EXTERNAL_RESOURCE}} : attaque par épuisement de ressource.

## Règles

- Tout endpoint POST à effet → quota par `{{IDENTIFIER_TYPE}}` (utilisateur, API key, IP).
- Limites minimales : définies par {{TECH_LEAD}} selon le SLO de l'endpoint.
- Dépassement → `429 Too Many Requests` + header `Retry-After: <secondes>` + corps d'erreur structuré (fiche A-03).
- Fallback quota : si le store de comptage est indisponible → fail-open avec alerte (ne pas bloquer la production).
- Métriques : `rate_limit_exceeded_total{endpoint, identifier}` exposée.

## Anti-pattern
```java
// ❌ BLOCAGE — endpoint à effet sans quota
@PostMapping("/{{ENDPOINT}}")
public {{RESPONSE}} create(@RequestBody @Valid {{COMMAND}} cmd) {
    return service.execute(cmd); // 0 protection contre l'inondation
}
```

## Pattern conforme
```java
// ✅ Rate limiter par identifiant
@PostMapping("/{{ENDPOINT}}")
@RateLimiter(name = "{{ENDPOINT_RATE_LIMITER}}")
public {{RESPONSE}} create(
        @AuthenticationPrincipal {{PRINCIPAL_TYPE}} principal,
        @RequestBody @Valid {{COMMAND}} cmd) {
    return service.execute(cmd);
}
```

```yaml
# application.yml
resilience4j.ratelimiter:
  instances:
    {{ENDPOINT_RATE_LIMITER}}:
      limitForPeriod: {{LIMIT}}
      limitRefreshPeriod: {{PERIOD}}
      timeoutDuration: 0s
```

```java
// ✅ Réponse 429 structurée
@ExceptionHandler(RequestNotPermitted.class)
ResponseEntity<ProblemDetail> handleRateLimit(RequestNotPermitted ex) {
    ProblemDetail pd = ProblemDetail.forStatus(429);
    pd.setTitle("Too Many Requests");
    pd.setDetail("Rate limit exceeded for {{ENDPOINT}}. Retry after " + retryAfter + "s.");
    return ResponseEntity.status(429)
        .header("Retry-After", String.valueOf(retryAfter))
        .body(pd);
}
```

## Checklist CI/CD
- [ ] Tout `POST` à effet → `@RateLimiter` ou filtre configuré
- [ ] Test : {{LIMIT}} + 1 appels en {{PERIOD}} → le dernier retourne `429` avec `Retry-After`
- [ ] Métrique `rate_limit_exceeded_total` visible dans le système de monitoring
- [ ] Alerte si taux de 429 > {{ALERT_THRESHOLD}} % sur {{ENDPOINT}} (signe d'attaque ou bug client)
