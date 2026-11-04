---
id: "F-03"
titre: "Mobile — Secure Storage & Sync offline idempotente"
severite: "🔴 BLOCAGE (donnée sensible en stockage non chiffré / sync sans idempotence)"
applicable_si: [mobile]
declencheurs:
  - "Toute donnée sensible stockée côté client dans {{MODULE_CIBLE}} : {{SENSITIVE_DATA_TYPES}}"
  - "Toute action à effet initiée hors-ligne"
  - "Toute sync de données vers le serveur"
version: "1.0"
---

## Risque
`{{SENSITIVE_DATA_TYPES}}` dans `{{INSECURE_STORAGE}}` (non chiffré) = lisible sur device rooté → hijack de session.
Sync à effet sans `{{IDEMPOTENCY_HEADER}}` + réseau instable → double delivery → double effet.

## Règles

**Stockage sécurisé :**
- Données sensibles → `{{SECURE_STORAGE_LIB}}` (Keychain iOS / Android Keystore) uniquement.
- Jamais : `{{INSECURE_STORAGE}}`, stockage local non chiffré pour des données sensibles.
- Base locale pour l'outbox → SQLite chiffré (`{{ENCRYPTED_DB_LIB}}`).

**Sync offline & idempotence :**
- Action à effet hors-ligne → **outbox persistée chiffrée** avec `{{IDEMPOTENCY_HEADER}}` généré à la consignation.
- État affiché : « en attente » — le succès n'est affiché qu'après **confirmation serveur**.
- Sync → même `{{IDEMPOTENCY_HEADER}}`, backoff + jitter + limite, serveur source de vérité.
- Conflit → le serveur gagne, le client re-synchronise et alerte l'utilisateur.

## Anti-pattern
```dart
// ❌ BLOCAGE — donnée sensible dans stockage non chiffré
final prefs = await {{INSECURE_STORAGE}}.getInstance();
await prefs.setString('{{SENSITIVE_KEY}}', sensitiveValue);

// ❌ BLOCAGE — sync sans idempotence
Future<void> syncPending() async {
  for (final op in pending) {
    await api.post('/{{ENDPOINT}}', op.toJson()); // chaque rejeu = effet potentiel
  }
}
```

## Pattern conforme
```dart
// ✅ Stockage sécurisé
final storage = const {{SECURE_STORAGE_LIB}}();
await storage.write(key: '{{SENSITIVE_KEY}}', value: sensitiveValue);
// → Keychain iOS / Android Keystore AES-256

// ✅ Outbox persistée avec {{IDEMPOTENCY_HEADER}}
Future<void> enqueue({{COMMAND}} cmd) async {
    final entry = OutboxEntry(
        id: Uuid().v4(),          // {{IDEMPOTENCY_HEADER}} généré à la consignation
        payload: cmd.toJson(),
        status: OutboxStatus.queued,
    );
    await outboxDb.insert(entry); // BDD chiffrée
    uiState = '{{PENDING_MESSAGE}}'; // jamais "succès" avant confirmation
}

// ✅ Sync idempotente — même clé, backoff, serveur source de vérité
Future<void> syncOutbox() async {
    final pending = await outboxDb.fetchQueued();
    for (final entry in pending) {
        try {
            final resp = await apiWithBackoff.post('/{{ENDPOINT}}',
                headers: {'{{IDEMPOTENCY_HEADER}}': entry.id}, // même clé
                body: entry.payload,
            );
            if (resp.isSuccess || resp.statusCode == 409) {
                await outboxDb.markSent(entry.id);
            }
        } catch (e) { /* retry géré par backoff */ }
    }
}
```

## Checklist CI/CD
- [ ] `grep -r "{{INSECURE_STORAGE}}" lib/ | grep -i "{{SENSITIVE_GREP_PATTERN}}"` → 0
- [ ] Test : action hors-ligne → kill app → relance → 1 seul envoi (même `{{IDEMPOTENCY_HEADER}}`)
- [ ] Test : coupure réseau pendant sync → rejeu → 1 effet idempotent
- [ ] Test : conflit de données → serveur gagne, UI re-synchronise et alerte
