---
id: "A-01"
titre: "Clean Architecture — Isolation domaine, SRP/DIP & Ubiquitous Language"
severite: "🔴 BLOCAGE (règle métier dans un adaptateur / import framework dans le domaine) — 🟡 ALERTE"
applicable_si: [backend]
declencheurs:
  - "Toute classe mélangeant règle métier et framework dans {{MODULE_CIBLE}}"
  - "Tout import framework dans le package domaine"
  - "Toute classe faisant > 3 choses distinctes (SRP violé)"
  - "Tout identifiant cryptique ou hors du glossaire métier {{DOMAIN_GLOSSARY}}"
version: "1.0"
---

## Risque
Règle métier dans un contrôleur : test requiert un contexte d'intégration (lent, fragile), un changement de framework casse la règle.
Nom cryptique (`trx`, `amt`) : un développeur met {{ONBOARDING_TIME}} à comprendre le modèle → erreur d'interprétation → bug métier.

## Règles

**Isolation domaine :**
- Règle métier ({{DOMAIN_RULES_EXAMPLES}}) → package `domain.*` sans aucun import framework (`{{FRAMEWORK_PACKAGES}}`).
- Adaptateurs (Controller, Listener, Repository) → appellent la règle, jamais l'inverse.
- Test de règle → 0 `{{INTEGRATION_TEST_ANNOTATION}}` (testable pur JUnit).

**SRP / DIP :**
- Une classe = une responsabilité (≤ 3 responsabilités, ≤ 300 lignes).
- Dépendre d'abstractions : `{{GATEWAY_INTERFACE}}` (interface) jamais `{{GATEWAY_IMPL}}` (concrète).
- Test avec > 3 mocks → signe SRP violé.

**Ubiquitous Language :**
- Glossaire obligatoire : `{{DOMAIN_TERMS}}`.
- Interdit : `Manager`, `Helper`, `Util` sans suffixe précis, identifiants abrégés.
- Convention casse : `camelCase` (variables), `PascalCase` (classes), `UPPER_SNAKE_CASE` (constantes).

## Anti-pattern
```java
// ❌ BLOCAGE — règle métier dans l'adaptateur
@RestController
class {{CONTROLLER}} {
    @PostMapping("/{{ENDPOINT}}")
    {{RESPONSE}} handle(@RequestBody {{COMMAND}} cmd) {
        if (cmd.amount().compareTo({{LIMIT}}) > 0) // règle dans l'adaptateur !
            throw new BusinessException("Refused");
        return service.process(cmd);
    }
}

// ❌ ALERTE — nom cryptique + SRP violé
class {{CRYPTIC_NAME}}Svc {
    void calc{{CRYPTIC_FIELD}}(Object usr, Object trx) { ... }
}
```

## Pattern conforme
```java
// ✅ Règle dans le domaine
// domain/{{DOMAIN_PACKAGE}}/{{RULE_CLASS}}.java — 0 import framework
public record {{RULE_CLASS}}(BigDecimal limit) {
    public void validate(BigDecimal amount) {
        if (amount.compareTo(limit) > 0)
            throw new {{RULE_EXCEPTION}}(amount, limit);
    }
}

// adapter/http/{{CONTROLLER}}.java
@RestController
class {{CONTROLLER}} {
    private final {{RULE_CLASS}} rule;        // dépendance au domaine
    private final {{GATEWAY_INTERFACE}} gw;   // abstraction

    @PostMapping("/{{ENDPOINT}}")
    {{RESPONSE}} handle(@RequestBody @Valid {{COMMAND}} cmd) {
        rule.validate(cmd.amount());           // règle depuis l'adaptateur
        return gw.process(cmd);
    }
}
```

## Checklist CI/CD
- [ ] ArchUnit : `noClasses().that().resideInPackage("..domain..")` → `dependOnClassesThat().resideInPackage("{{FRAMEWORK_PACKAGES}}")`
- [ ] Test de règle `{{RULE_CLASS}}Test` → 0 `{{INTEGRATION_TEST_ANNOTATION}}`, exécution < 100 ms
- [ ] `grep -r "import {{FRAMEWORK_PACKAGES}}" src/main/java/**/domain/` → 0
- [ ] `grep -r "class.*Manager\|class.*Helper\|class.*Util" src/ | grep -v "//"` → revue manuelle
