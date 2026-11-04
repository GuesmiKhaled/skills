---
id: "T-02"
titre: "Complexité & Lisibilité — Cyclomatique ≤ {{CYCLO_THRESHOLD}}, Cognitive ≤ {{COGNITIVE_THRESHOLD}}, zéro magic number"
severite: "🟡 ALERTE (complexité hors seuil / magic number dans règle métier)"
applicable_si: [backend, frontend]
declencheurs:
  - "Toute méthode dépassant la complexité cyclomatique ou cognitive définie"
  - "Tout magic number ou chaîne magique dans le code (sauf 0 et 1)"
  - "SonarQube S3776 (Cognitive Complexity) ou S1479 (Switch > N cases)"
version: "1.0"
---

## Risque
Complexité cognitive {{COGNITIVE_DANGER}} dans `{{COMPLEX_METHOD}}` : un développeur passe {{COMPREHENSION_TIME}} à comprendre le flux, introduit un bug en modifiant la mauvaise branche → régression.
`if (status == {{MAGIC_NUMBER}})` dans la règle de {{DOMAIN_RULE}} : que vaut {{MAGIC_NUMBER}} ? Un refactoring silencieux change la signification → bug métier.

## Règles

**Complexité :**
- Cyclomatique ≤ {{CYCLO_THRESHOLD}} par méthode (Sonar S1541) — au-delà : extraire ou simplifier.
- Cognitive ≤ {{COGNITIVE_THRESHOLD}} par méthode (Sonar S3776) — l'imbrication compte plus que le branchement.
- Méthode > {{MAX_METHOD_LINES}} lignes → ALERTE : candidat à la décomposition.
- Classe > {{MAX_CLASS_LINES}} lignes → ALERTE : SRP probablement violé (fiche A-01).

**Magic numbers & chaînes :**
- Tout littéral numérique (sauf `0` et `1`) dans une règle métier → constante nommée.
- Toute chaîne répétée (code, statut) → constante ou enum.
- Constantes métier → classe `{{CONSTANTS_CLASS}}` ou enum `{{STATUS_ENUM}}`, `{{CODE_ENUM}}`.

## Anti-pattern
```java
// ❌ ALERTE Sonar S3776 — complexité cognitive > {{COGNITIVE_THRESHOLD}}
public {{RESULT}} processOperation({{ENTITY}} e) {
    if (e.getStatus() == 2) {                          // +1 : que vaut 2 ?
        if (e.getAmount().compareTo(new BigDecimal("{{MAGIC_THRESHOLD}}")) > 0) { // +2 magic
            for ({{ITEM}} item : e.getItems()) {        // +3
                if (item.getCode().equals("{{MAGIC_CODE}}")) { // +4 magic string
                    if (!item.isExpired()) {            // +5
                        // ...
                    }
                }
            }
        }
    }
}

// ❌ ALERTE — magic numbers dans règle métier
if (responseCode == {{MAGIC_CODE_VALUE}}) { reject(); }
if (attempts > {{MAGIC_MAX_ATTEMPTS}}) { block(); }
```

## Pattern conforme
```java
// ✅ Constantes nommées + enum
public enum {{STATUS_ENUM}} {
    {{STATUS_VALUES}};
}

public enum {{CODE_ENUM}} {
    {{CODE_APPROVED}}("{{CODE_APPROVED_VALUE}}"),
    {{CODE_REJECTED}}("{{CODE_REJECTED_VALUE}}");
    private final String code;
    {{CODE_ENUM}}(String code) { this.code = code; }
    public static {{CODE_ENUM}} of(String code) {
        return Arrays.stream(values()).filter(c -> c.code.equals(code)).findFirst()
            .orElse({{CODE_REJECTED}});
    }
}

public final class {{CONSTANTS_CLASS}} {
    public static final BigDecimal HIGH_RISK_THRESHOLD = new BigDecimal("{{MAGIC_THRESHOLD}}");
    public static final int        MAX_ATTEMPTS = {{MAGIC_MAX_ATTEMPTS}};
    private {{CONSTANTS_CLASS}}() {}
}

// ✅ Méthode décomposée — complexité cognitive ≤ {{COGNITIVE_THRESHOLD}}
public {{RESULT}} processOperation({{ENTITY}} e) {
    validateState(e);
    {{ITEM}} item = findValidItem(e);
    return process(e, item);
}

private void validateState({{ENTITY}} e) {
    if (e.getStatus() != {{STATUS_ENUM}}.{{EXPECTED_STATUS}})
        throw new InvalidStateException(e.getId(), e.getStatus());
}
```

## Checklist CI/CD
- [ ] Sonar quality gate : cyclomatique ≤ {{CYCLO_THRESHOLD}}, cognitive ≤ {{COGNITIVE_THRESHOLD}} → build bloqué si dépassé
- [ ] `grep -r "== [2-9][0-9]\+\|== [0-9][0-9][0-9]" src/main/` → revue manuelle (magic numbers)
- [ ] `grep -r "\.equals(\"[0-9][0-9]\")" src/ | grep -i "code\|status"` → remplacer par enum {{CODE_ENUM}}
- [ ] Méthode > {{MAX_METHOD_LINES}} lignes → justification requise dans la PR
