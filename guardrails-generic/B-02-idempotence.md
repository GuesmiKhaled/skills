---
id: "B-02"
titre: "Idempotence des opérations à effet"
severite: "🔴 BLOCAGE absolu"
applicable_si: [backend]
declencheurs:
  - "Tout POST créant un effet : {{OPERATION_TYPES}} (création, modification d'état, envoi)"
  - "Tout handler de message traitant un événement à effet dans {{MODULE_CIBLE}}"
version: "1.0"
---

## Risque
Sans idempotence : un retry réseau, un double-envoi ou un double-delivery de message = double effet.
La seule protection fiable : vérifier si l'opération a **déjà** été exécutée **avant** de l'exécuter.

## Règles

- Tout endpoint POST à effet → en-tête `{{IDEMPOTENCY_HEADER}}` obligatoire (UUID v4 client).
- Avant tout effet : lookup `{{IDEMPOTENCY_TABLE}} WHERE key = :key AND scope = :scope`.
  - Trouvé → retourner **exactement** le même résultat stocké (pas de 2e exécution).
  - Absent → exécuter + persister (key, scope, résultat, expiry) dans la **même transaction**.
- Retry → **même** `{{IDEMPOTENCY_HEADER}}` que la tentative originale — jamais une nouvelle clé.
- Consumer de messages → clé d'idempotence = identifiant stable du message.

## Anti-pattern
```java
// ❌ BLOCAGE — POST à effet sans clé d'idempotence
@PostMapping("/{{ENDPOINT}}")
public {{RESPONSE}} create(@RequestBody @Valid {{COMMAND}} cmd) {
    return service.execute(cmd); // rejouable = double effet
}

// ❌ BLOCAGE — retry avec nouvelle clé
for (int i = 0; i < 3; i++) {
    cmd = cmd.withIdempotencyKey(UUID.randomUUID()); // nouvelle clé = nouvel effet
    client.post(cmd);
}
```

## Pattern conforme
```java
// ✅ Endpoint avec clé d'idempotence
@PostMapping("/{{ENDPOINT}}")
public ResponseEntity<{{RESPONSE}}> create(
        @RequestHeader("{{IDEMPOTENCY_HEADER}}") UUID idempotencyKey,
        @RequestBody @Valid {{COMMAND}} cmd) {

    return idempotencyStore.find(idempotencyKey, "{{SCOPE}}")
        .map(stored -> ResponseEntity.ok(stored.result()))
        .orElseGet(() -> {
            {{RESPONSE}} result = service.execute(cmd);
            idempotencyStore.store(idempotencyKey, "{{SCOPE}}", result, Duration.ofHours(24));
            return ResponseEntity.status(201).body(result);
        });
}

// ✅ Retry — même clé originale
UUID originalKey = UUID.randomUUID(); // générée une seule fois
for (int attempt = 0; attempt < 6; attempt++) {
    cmd = cmd.withIdempotencyKey(originalKey); // MÊME clé
    try { client.post(cmd); break; }
    catch (RetryableException e) { backoff(attempt); }
}
```

## Checklist CI/CD
- [ ] Tout `@PostMapping` à effet → paramètre `@RequestHeader("{{IDEMPOTENCY_HEADER}}")` présent
- [ ] Test : même clé envoyée 2 fois → réponse identique, 1 seul enregistrement en BDD
- [ ] Test : consumer rejoué (même message) → 1 seul effet en BDD
- [ ] Index unique sur `(key, scope)` dans la table `{{IDEMPOTENCY_TABLE}}`
