---
id: "A-04"
titre: "Tracing distribué — correlationId & propagation inter-services"
severite: "🔴 BLOCAGE (log sur flux critique sans correlationId) — 🟡 ALERTE"
applicable_si: [backend]
declencheurs:
  - "Tout endpoint qui initie un flux multi-services dans {{MODULE_CIBLE}}"
  - "Tout appel inter-services (HTTP, messaging) sans propagation de correlationId"
  - "Tout log sur un flux critique sans identifiant de corrélation"
version: "1.0"
---

## Risque
Sans correlationId : un incident P1 en prod → recherche du flux dans les logs → impossible de relier les logs de {{SERVICE_COUNT}} services différents → debug impossible sans corrélation.
MDC non propagé aux threads de messaging → le correlationId disparaît entre les services → traçabilité perdue.

## Règles

- Tout endpoint entrant → `correlationId` dans le MDC en début de traitement.
- Priorité : en-tête `{{CORRELATION_HEADER}}` entrant → sinon UUID v4 généré.
- Propagé : appels HTTP sortants (en-tête `{{CORRELATION_HEADER}}`) + messages (header `{{CORRELATION_HEADER}}`).
- MDC nettoyé en fin de requête (`try/finally`) — sinon fuite entre requêtes (pool de threads).
- Format de log : `correlationId` présent dans chaque ligne du flux critique.

## Anti-pattern
```java
// ❌ BLOCAGE — log critique sans correlationId
log.info("Operation completed for entity {}", entityId);
// → impossible de relier ce log à un flux spécifique lors d'un incident

// ❌ ALERTE — MDC non nettoyé (fuite dans le pool de threads)
MDC.put("correlationId", id);
// (pas de finally pour le retirer)
```

## Pattern conforme
```java
// ✅ Filtre MDC — intercepte toutes les requêtes
@Component
public class CorrelationIdFilter extends OncePerRequestFilter {
    @Override protected void doFilterInternal(HttpServletRequest req, ...) {
        String correlationId = Optional.ofNullable(req.getHeader("{{CORRELATION_HEADER}}"))
            .filter(s -> !s.isBlank())
            .orElse(UUID.randomUUID().toString());

        MDC.put("correlationId", correlationId);
        resp.setHeader("{{CORRELATION_HEADER}}", correlationId);
        try {
            chain.doFilter(req, resp);
        } finally {
            MDC.remove("correlationId"); // nettoyage obligatoire
        }
    }
}

// ✅ Propagation appel HTTP sortant
restClient.get()
    .header("{{CORRELATION_HEADER}}", MDC.get("correlationId"))
    .retrieve()...

// ✅ Propagation message
messagingClient.send(MessageBuilder
    .withPayload(event)
    .setHeader("{{CORRELATION_HEADER}}", MDC.get("correlationId"))
    .build());

// ✅ Consumer de messages — restauration du MDC
void handle({{MESSAGE_TYPE}} msg, {{ACK_TYPE}} ack) {
    String corrId = extractCorrelationId(msg).orElse(UUID.randomUUID().toString());
    MDC.put("correlationId", corrId);
    try {
        processMessage(msg);
    } finally {
        MDC.remove("correlationId");
    }
    ack.acknowledge();
}
```

```
# Format de log — correlationId dans chaque ligne
{{LOG_PATTERN}} = %d{ISO8601} [%X{correlationId}] %-5level %logger - %msg%n
```

## Checklist CI/CD
- [ ] Appel à `{{EXAMPLE_ENDPOINT}}` → réponse avec header `{{CORRELATION_HEADER}}` présent
- [ ] Log du flux → chaque ligne contient `[<uuid>]` (MDC)
- [ ] Test : appel sans `{{CORRELATION_HEADER}}` → UUID généré et propagé à tous les services du flux
- [ ] `grep -r "MDC.put" src/ | grep -v "MDC.remove"` → vérifier que chaque `put` a son `remove` dans un `finally`
