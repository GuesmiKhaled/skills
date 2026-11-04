---
id: "D-04"
titre: "Pool de connexions & resource leak"
severite: "🔴 BLOCAGE (connexion/stream non fermé) — 🟡 ALERTE (pool mal dimensionné)"
applicable_si: [backend]
declencheurs:
  - "Tout code ouvrant une connexion BDD, un fichier, un stream ou une ressource réseau"
  - "Toute configuration de pool de connexions ({{CONNECTION_POOL_LIB}}) dans {{MODULE_CIBLE}}"
  - "SonarQube issue S2095 (Resource should be closed)"
version: "1.0"
---

## Risque
Une connexion non fermée dans un catch block → fuite silencieuse → le pool se vide progressivement → timeout en cascade sur tous les endpoints de {{MODULE_CIBLE}} → P1.
Pool sous-dimensionné → queue de connexions → latence p99 qui explose sous charge.

## Règles

- Tout resource (`Connection`, `ResultSet`, `Statement`, `InputStream`, `OutputStream`, `Channel`) → `try-with-resources`.
- Jamais de `.close()` manuel dans un `finally` séparé (oublié dans les exceptions).
- `{{CONNECTION_POOL_LIB}}` : `leakDetectionThreshold={{LEAK_DETECTION_THRESHOLD}}` en dev.
- Dimensionnement : `maximumPoolSize = {{POOL_SIZE_FORMULA}}` (ajusté par benchmark).
- `connectionTimeout ≤ {{CONNECTION_TIMEOUT}}` sur un endpoint à SLO défini.
- Jamais d'appel réseau/message **dans** une transaction BDD ouverte (verrou détenu le temps de l'appel).

## Anti-pattern
```java
// ❌ BLOCAGE S2095 — connexion non fermée si exception
Connection conn = dataSource.getConnection();
PreparedStatement ps = conn.prepareStatement(sql);
ResultSet rs = ps.executeQuery();
// Si rs.next() lève une exception → conn jamais fermée → fuite

// ❌ ALERTE — appel réseau dans @Transactional
@Transactional
public {{RESULT}} process({{ID}} id) {
    {{ENTITY}} e = repository.findById(id); // connexion + verrou
    {{RESPONSE}} r = {{EXTERNAL_CLIENT}}.call(e); // appel réseau {{EXTERNAL_TIMEOUT}} → verrou détenu ∞
    e.setResult(r.getValue());
    return repository.save(e);
}
```

## Pattern conforme
```java
// ✅ try-with-resources — fermeture garantie
try (Connection conn = dataSource.getConnection();
     PreparedStatement ps = conn.prepareStatement(sql);
     ResultSet rs = ps.executeQuery()) {
    while (rs.next()) { /* traitement */ }
} // conn, ps, rs fermés automatiquement même en cas d'exception

// ✅ Appel réseau HORS transaction — transaction courte
public {{RESULT}} process({{ID}} id) {
    {{ENTITY}} e = repository.findById(id);          // lecture hors transaction longue
    {{RESPONSE}} r = {{EXTERNAL_CLIENT}}.call(e);    // appel réseau hors transaction
    return persistResult(id, r);                      // transaction courte
}

@Transactional(timeout = {{TRANSACTION_TIMEOUT}})
{{RESULT}} persistResult({{ID}} id, {{RESPONSE}} r) {
    {{ENTITY}} e = repository.findById(id);
    e.setResult(r.getValue());
    return mapper.toResult(repository.save(e));
}
```

```yaml
# ✅ Pool dimensionné — {{CONNECTION_POOL_LIB}}
datasource:
  hikari:
    maximum-pool-size: {{MAX_POOL_SIZE}}
    minimum-idle: {{MIN_IDLE}}
    connection-timeout: {{CONNECTION_TIMEOUT}}
    leak-detection-threshold: {{LEAK_DETECTION_THRESHOLD}}
```

## Checklist CI/CD
- [ ] Sonar S2095 → 0 issue ouverte sur les classes d'accès BDD de {{MODULE_CIBLE}}
- [ ] `grep -r "\.getConnection()" src/ | grep -v "try ("` → 0
- [ ] Test de charge {{LOAD_DURATION}} : pool `ActiveConnections` ne monte pas indéfiniment
- [ ] `leakDetectionThreshold` activé en staging → 0 alerte de fuite
- [ ] `connectionTimeout={{CONNECTION_TIMEOUT}}` → test : BDD down → endpoint répond en < {{TIMEOUT_RESPONSE}} ms
