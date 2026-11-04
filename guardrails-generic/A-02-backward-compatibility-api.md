---
id: "A-02"
titre: "Backward Compatibility API — zéro breaking change sans versioning"
severite: "🔴 BLOCAGE (breaking change sur API en production)"
applicable_si: [backend, frontend]
declencheurs:
  - "Toute modification de contrat API dans {{MODULE_CIBLE}} : suppression/renommage/changement de type"
  - "Tout nouveau endpoint ou nouvelle version d'un endpoint existant"
  - "Toute dépréciation d'un champ ou endpoint utilisé en production"
version: "1.0"
---

## Risque
Un champ supprimé ou renommé dans une API utilisée en production → erreur de désérialisation dès le déploiement → incident client.
Un changement de type (`string` → `int`) sur un champ → perte silencieuse de données côté consommateur.

## Règles

**Breaking changes interdits sans versioning :**
- Supprimer un champ → BLOCAGE : déprécier d'abord, supprimer après {{DEPRECATION_PERIOD}}.
- Renommer un champ → BLOCAGE : ajouter le nouveau nom, garder l'ancien (déprécié).
- Changer le type d'un champ → BLOCAGE : nouveau champ + migration côté client.
- Rendre un champ `required` qui était optionnel → BLOCAGE.

**Évolutions non-breaking (autorisées) :**
- Ajouter un nouveau champ optionnel.
- Ajouter un nouveau endpoint.
- Ajouter une nouvelle valeur d'enum (si le client ignore les inconnues).

**Versioning :**
- Breaking change inévitable → versioning par path : `/v1/{{ENDPOINT}}` → `/v2/{{ENDPOINT}}`.
- `Sunset` header sur les endpoints v1 dépréciés (RFC 8594) : `Sunset: {{SUNSET_DATE}}`.
- Maintenir au max {{MAX_PARALLEL_VERSIONS}} versions simultanées.

## Anti-pattern
```java
// ❌ BLOCAGE — champ renommé en place
public record {{RESPONSE_TYPE}}(
    UUID id,
    BigDecimal totalAmount,  // était "amount" — casse tous les clients existants
    String status
) {}
```

## Pattern conforme
```java
// ✅ Champ déprécié conservé + nouveau champ ajouté
public record {{RESPONSE_TYPE}}(
    UUID id,
    @Deprecated {{FIELD_TYPE}} {{OLD_FIELD}},  // maintenu pour les anciens clients
    {{FIELD_TYPE}} {{NEW_FIELD}},              // nouveau nom — parallèle
    String status
) {}

// ✅ Versioning par path
@RestController @RequestMapping("/v2/{{ENDPOINT}}")
class {{CONTROLLER_V2}} { /* Nouvelle implémentation */ }

// ✅ Sunset header sur v1
@GetMapping("/v1/{{ENDPOINT}}/{id}")
ResponseEntity<{{RESPONSE_V1}}> getV1(...) {
    return ResponseEntity.ok()
        .header("Sunset", "{{SUNSET_DATE}}")
        .header("Deprecation", "true")
        .body(serviceV1.find(id));
}
```

## Checklist CI/CD
- [ ] Consumer-driven contract tests (Pact ou équivalent) → contrats passent avant le merge
- [ ] `grep -r "@Deprecated" src/ | grep -v "// .*removed"` → chaque `@Deprecated` a une date de suppression
- [ ] Nouveau endpoint → spec OpenAPI mise à jour + revue par un consommateur
- [ ] `Sunset` header présent sur tout endpoint v1 déprécié
- [ ] Aucun champ supprimé sans période de dépréciation de {{DEPRECATION_PERIOD}}
