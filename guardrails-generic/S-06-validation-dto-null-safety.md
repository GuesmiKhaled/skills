---
id: "S-06"
titre: "Défense en profondeur — validation DTO d'entrée + null safety"
severite: "🔴 BLOCAGE (DTO critique sans @Valid / return null sur méthode publique)"
applicable_si: [backend]
declencheurs:
  - "Tout @RequestBody sur un endpoint à effet dans {{MODULE_CIBLE}}"
  - "Tout DTO/Command passé à un service métier sans contraintes de validation"
  - "Tout return null sur une méthode publique"
  - "Tout Optional.get() sans isPresent()/orElse()"
version: "1.0"
---

## Risque
Un DTO sans `@Valid` : une valeur `null` atteint le domaine → NPE au cœur de la règle métier, état incohérent.
Un `return null` sur une méthode publique : le NPE se propage silencieusement plusieurs couches plus loin.

## Règles

**Validation d'entrée (couche Controller) :**
- Tout `@RequestBody` sur endpoint à effet → `@Valid` obligatoire.
- Tout DTO/Command → contraintes Bean Validation : `@NotNull`, `@NotBlank`, `@Size`, `@Min`, `@Max`.
- Violation → handler d'erreur avec format standardisé (fiche A-03).

**Null safety (couche domaine) :**
- Méthode publique retournant une entité optionnelle → `Optional<T>`, jamais `null`.
- Collection vide → `List.of()` / `Collections.emptyList()`, jamais `null`.
- `Optional.get()` → toujours `.orElse()`, `.orElseThrow()` ou `.ifPresent()`.
- Paramètre critique → `Objects.requireNonNull(param, "message")` en début de méthode.

## Anti-pattern
```java
// ❌ BLOCAGE — DTO sans validation : valeur null atteint le domaine
@PostMapping("/{{ENDPOINT}}")
public {{RESPONSE}} handle(@RequestBody {{COMMAND}} cmd) { ... }

record {{COMMAND}}(String userId, BigDecimal amount) {} // pas de contrainte

// ❌ BLOCAGE — return null sur méthode publique
public {{ENTITY}} findById(UUID id) {
    return repository.findById(id).orElse(null); // NPE plusieurs couches plus loin
}
```

## Pattern conforme
```java
// ✅ Endpoint avec @Valid
@PostMapping("/{{ENDPOINT}}")
public {{RESPONSE}} handle(@RequestBody @Valid {{COMMAND}} cmd) { ... }

// ✅ DTO avec contraintes
record {{COMMAND}}(
    @NotBlank  String userId,
    @NotNull @DecimalMin("0.01") BigDecimal amount,
    @NotBlank  String idempotencyKey
) {
    {{COMMAND}} {
        Objects.requireNonNull(userId, "userId is required");
        if (amount.compareTo(BigDecimal.ZERO) <= 0)
            throw new IllegalArgumentException("amount must be > 0");
    }
}

// ✅ Optional sur méthode publique
public Optional<{{ENTITY}}> findById(UUID id) {
    return repository.findById(id);
}

// ✅ Utilisation correcte
{{ENTITY}} entity = repo.findById(id)
    .orElseThrow(() -> new NotFoundException(id));
```

## Checklist CI/CD
- [ ] `grep -r "@RequestBody" src/ | grep -v "@Valid"` → 0 résultats sur endpoints à effet
- [ ] `grep -r "return null" src/main/` → 0 sur méthodes publiques de service/domaine
- [ ] Test : POST avec champ `amount=null` → `400 Bad Request` avec détail structuré
- [ ] Test : DTO avec valeur invalide → exception dans le constructeur compact
