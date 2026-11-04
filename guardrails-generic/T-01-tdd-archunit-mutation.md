---
id: "T-01"
titre: "TDD, ArchUnit & Mutation Testing — la qualité des tests est prouvable"
severite: "🔴 BLOCAGE (règle architecture sans test ArchUnit) — 🟡 ALERTE (mutant survivant sur règle critique)"
applicable_si: [backend]
declencheurs:
  - "Toute PR sur une règle critique ({{CRITICAL_RULES}}) avec couverture haute mais assertions faibles"
  - "Tout import framework dans le domaine non bloqué par ArchUnit"
  - "Toute dépendance circulaire non détectée en CI"
  - "Score mutation < {{MUTATION_THRESHOLD}} % sur une classe du cœur ({{CRITICAL_CLASSES}})"
version: "1.0"
---

## Risque
100 % de couverture ne prouve rien : un test sans assertion passe, le mutant `<` → `<=` sur la règle de `{{BOUNDARY_RULE}}` survit → bug en prod découvert par le client.
Sans ArchUnit : la règle « domaine sans framework » est un vœu pieux — un import passe silencieusement en CI et crée un couplage irréversible.
Dépendance circulaire ({{CIRCULAR_DEP_EXAMPLE}}) non détectée → refactoring impossible.

## Règles

**TDD :**
- Rouge → Vert → Refactor sur toute règle métier ({{DOMAIN_RULES_EXAMPLES}}).
- Test unitaire de règle : 0 `{{INTEGRATION_TEST_ANNOTATION}}`, exécution < 100 ms.
- Assertion sur la valeur exacte : `assertThat(result).isEqualByComparingTo("{{EXPECTED_VALUE}}")`.

**ArchUnit — 4 règles obligatoires en CI :**
```java
// Règle 1 : le domaine ne dépend d'aucun framework
noClasses().that().resideInPackage("..domain..")
    .should().dependOnClassesThat()
    .resideInAnyPackage("{{FRAMEWORK_PACKAGES}}")
    .check(importedClasses);

// Règle 2 : pas de cycle entre packages
SlicesRuleDefinition.slices()
    .matching("{{BASE_PACKAGE}}.(*)..")
    .should().beFreeOfCycles()
    .check(importedClasses);

// Règle 3 : layering obligatoire (controller → service → repository)
layeredArchitecture()
    .consideringAllDependencies()
    .layer("Controller").definedBy("..controller..")
    .layer("Service")   .definedBy("..service..")
    .layer("Repository").definedBy("..repository..")
    .whereLayer("Controller").mayOnlyBeAccessedByLayers("Service")
    .whereLayer("Repository").mayOnlyBeAccessedByLayers("Service")
    .check(importedClasses);

// Règle 4 : pas de logique métier dans un repository
noClasses().that().resideInPackage("..repository..")
    .should().haveSimpleNameContaining("Service")
    .orShould().haveSimpleNameContaining("Rule")
    .check(importedClasses);
```

**Mutation Testing ({{MUTATION_TOOL}}) — classes critiques :**
- Score minimum : {{MUTATION_THRESHOLD}} % (seuil CI bloquant), cible {{MUTATION_TARGET}} % sur `{{CRITICAL_CLASSES}}`.
- Mutant survivant sur règle d'argent/bordure → renforcer le test avec valeur exacte de bordure.

## Anti-pattern
```java
// ❌ ALERTE — test sans assertion
@Test void testProcess() {
    service.process(command); // pas d'assertion → n'importe quel bug passe
}

// ❌ ALERTE — couverture sans preuve de bordure
@Test void testValidation() {
    assertDoesNotThrow(() -> rule.validate(new BigDecimal("{{BOUNDARY_VALUE}}")));
    // montant exact à la limite jamais testé → mutant < → <= survit
}
```

## Pattern conforme
```java
// ✅ Test rouge-vert-refactor avec assertion sur valeur exacte
@Test void should_calculate_{{RULE_NAME}}_correctly() {
    BigDecimal input    = new BigDecimal("{{INPUT_VALUE}}");
    BigDecimal expected = new BigDecimal("{{EXPECTED_VALUE}}");
    BigDecimal result   = calculator.calculate(input);
    assertThat(result).isEqualByComparingTo(expected); // tue le mutant ±0.01
}

// ✅ Bordure exacte — tue le mutant < → <=
@Test void should_reject_{{RULE_NAME}}_equal_to_limit() {
    BigDecimal limit  = new BigDecimal("{{LIMIT_VALUE}}");
    BigDecimal tested = new BigDecimal("{{LIMIT_VALUE}}"); // exactement la limite
    assertThatThrownBy(() -> rule.validate(tested, limit))
        .isInstanceOf({{LIMIT_EXCEPTION}}.class);
}

// ✅ ArchUnit dans la suite CI
@AnalyzeClasses(packages = "{{BASE_PACKAGE}}")
class ArchitectureRulesTest implements ArchRuleDefinition {
    // Les 4 règles ci-dessus en @ArchTest
}
```

```xml
<!-- pom.xml — {{MUTATION_TOOL}} seuil bloquant en CI -->
<plugin>
    <groupId>org.pitest</groupId>
    <artifactId>pitest-maven</artifactId>
    <configuration>
        <targetClasses>
            <param>{{BASE_PACKAGE}}.*.domain.*</param>
        </targetClasses>
        <mutationThreshold>{{MUTATION_THRESHOLD}}</mutationThreshold>
        <coverageThreshold>{{COVERAGE_THRESHOLD}}</coverageThreshold>
    </configuration>
</plugin>
```

## Checklist CI/CD
- [ ] `mvn archunit:check` (ou équivalent) → 0 violation (4 règles) — bloque le build en CI
- [ ] `{{MUTATION_TOOL}}` → score ≥ {{MUTATION_THRESHOLD}} % sur `domain.*` — bloque si < seuil
- [ ] Aucun mutant survivant sur `{{CRITICAL_CLASSES}}` (rapport {{MUTATION_TOOL}})
- [ ] Test de dépendance circulaire → règle `beFreeOfCycles` verte
- [ ] Test de règle métier : 0 `{{INTEGRATION_TEST_ANNOTATION}}` sur les tests de domaine
- [ ] `grep -r "assertDoesNotThrow\|assertTrue(true)" src/test/` → revue manuelle
