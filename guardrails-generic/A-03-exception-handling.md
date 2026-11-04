---
id: "A-03"
titre: "Exception Handling — format structuré, zéro stack dans la réponse"
severite: "🔴 BLOCAGE (stack trace dans la réponse / exception avalée)"
applicable_si: [backend]
declencheurs:
  - "Tout endpoint pouvant retourner une erreur dans {{MODULE_CIBLE}}"
  - "Toute exception métier ou technique non mappée vers un code HTTP cohérent"
  - "Toute réponse d'erreur sans format standardisé"
version: "1.0"
---

## Risque
Stack trace dans la réponse HTTP → fuite du nom des classes, packages, versions → surface d'attaque.
Sans format standardisé → le front ne peut pas distinguer une 4xx (définitive) d'une 5xx (rejouable) → l'utilisateur reclique → double action.

## Règles

- Tout endpoint → handler global d'exceptions qui intercepte **toutes** les exceptions.
- Format obligatoire : `{{ERROR_FORMAT}}` (RFC 7807 ou équivalent) avec :
  - `type` / `code` : identifiant du type d'erreur.
  - `title` / `message` : message humain court.
  - `status` : code HTTP.
  - `detail` : explication sans information technique interne.
- 4xx → erreur client définitive, non rejouable. Stack jamais dans la réponse.
- 5xx → erreur technique potentiellement rejouable. Stack dans les **logs**, pas dans la réponse.
- `catch (Exception e) {}` vide → BLOCAGE absolu.

## Anti-pattern
```java
// ❌ BLOCAGE — stack trace dans la réponse (comportement Spring Boot par défaut non configuré)
@GetMapping("/{{ENDPOINT}}/{id}")
{{ENTITY}} get(@PathVariable {{ID_TYPE}} id) throws Exception {
    return service.find(id); // stack exposée si exception
}

// ❌ BLOCAGE — exception avalée
try {
    {{EXTERNAL_SERVICE}}.process(id);
} catch (Exception e) {
    // rien — état incohérent silencieux
}
```

## Pattern conforme
```java
// ✅ Handler global — {{ERROR_FORMAT}}
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler({{ENTITY}}NotFoundException.class)
    {{ERROR_RESPONSE_TYPE}} handleNotFound({{ENTITY}}NotFoundException ex) {
        {{ERROR_RESPONSE_TYPE}} err = {{ERROR_RESPONSE_TYPE}}.forStatus(404);
        err.setType("{{ERROR_TYPE_BASE_URL}}/not-found");
        err.setTitle("{{ENTITY}} Not Found");
        err.setDetail("No {{ENTITY}} found with id: " + ex.getId()); // pas de stack
        err.setProperty("errorCode", "{{ENTITY_UPPER}}_NOT_FOUND");
        return err;
    }

    @ExceptionHandler({{BUSINESS_RULE_EXCEPTION}}.class)
    {{ERROR_RESPONSE_TYPE}} handleBusinessRule({{BUSINESS_RULE_EXCEPTION}} ex) {
        {{ERROR_RESPONSE_TYPE}} err = {{ERROR_RESPONSE_TYPE}}.forStatus(422);
        err.setTitle("Business Rule Violated");
        err.setDetail(ex.getPublicMessage()); // jamais ex.getMessage() brut
        err.setProperty("errorCode", ex.getErrorCode());
        return err; // 4xx → non rejouable
    }

    @ExceptionHandler({{TECHNICAL_EXCEPTION}}.class)
    {{ERROR_RESPONSE_TYPE}} handleTechnical({{TECHNICAL_EXCEPTION}} ex, HttpServletRequest req) {
        log.error("Technical error on {}", req.getRequestURI(), ex); // stack en LOG uniquement
        {{ERROR_RESPONSE_TYPE}} err = {{ERROR_RESPONSE_TYPE}}.forStatus(503);
        err.setTitle("Service Temporarily Unavailable");
        err.setDetail("Please retry in a few seconds."); // pas de stack
        err.setProperty("errorCode", "SERVICE_UNAVAILABLE");
        return err; // 5xx → rejouable
    }
}
```

## Checklist CI/CD
- [ ] `curl /{{ENDPOINT}}/nonexistent` → réponse JSON structurée (type, title, status, errorCode) — 0 stack
- [ ] `grep -r "catch.*Exception.*{$" src/ | grep -v "//"` → 0 catch vide
- [ ] Test : 500 → log contient la stack ; réponse HTTP ne contient pas `stackTrace`
- [ ] Test : 4xx → retryable=false ; 5xx → retryable=true (côté client front)
