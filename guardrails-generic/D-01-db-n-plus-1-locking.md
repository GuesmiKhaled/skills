---
id: "D-01"
titre: "BDD — Anti N+1, pagination keyset & locking concurrent"
severite: "🔴 BLOCAGE (N+1 hot path / update critique sans verrou) — 🟡 ALERTE"
applicable_si: [backend]
declencheurs:
  - "Tout repository/service chargeant une liste avec associations dans {{MODULE_CIBLE}}"
  - "Toute boucle qui fait un appel BDD par itération"
  - "Toute mise à jour d'une valeur critique ({{CRITICAL_FIELD_EXAMPLES}} : solde, compteur, quota)"
  - "Tout SELECT ... FOR UPDATE ou verrou explicite"
version: "1.0"
---

## Risque
N+1 sur {{TABLE}} avec {{ASSOCIATED_TABLES}} → {{N_PLUS_ONE_QUERIES}} requêtes par appel → pool de connexions saturé → timeout en cascade → SLO p99 dépassé.
`read-modify-write` en deux requêtes sur `{{CRITICAL_FIELD}}` → lost update : deux écritures concurrentes lisent la même valeur → écart comptable non justifiable.

## Règles

**Anti N+1 :**
- Liste avec associations → batch (une requête) — jamais boucle `findById`.
- `findAll()` sans limite → BLOCAGE : pagination keyset obligatoire.
- Keyset : `WHERE {{SORT_FIELD}} > :lastValue ORDER BY {{SORT_FIELD}} LIMIT :n` (pas OFFSET profond).
- Projection DTO (colonnes utiles) — jamais `SELECT *` sur liste avec champs lourds.

**Locking concurrent :**
- Mise à jour de `{{CRITICAL_FIELD}}` → **optimistic par défaut** (`@Version` + gestion exception).
- Décrément d'une ressource rare → requête atomique : `UPDATE {{TABLE}} SET {{FIELD}} = {{FIELD}} - :delta WHERE id = :id AND {{FIELD}} >= :delta`.
- `FOR UPDATE` : timeout + transaction courte (verrou libéré en quelques ms).
- Jamais de `read-modify-write` en deux étapes hors transaction.

## Anti-pattern
```java
// ❌ BLOCAGE — N+1
List<{{ENTITY}}> items = repository.findAll();
for ({{ENTITY}} item : items) {
    {{ASSOCIATED_ENTITY}} assoc = assocRepository.findById(item.getAssocId()); // +N requêtes
}

// ❌ BLOCAGE — lost update sur {{CRITICAL_FIELD}}
{{ENTITY}} e = repository.findById(id);
e.set{{CRITICAL_FIELD}}(e.get{{CRITICAL_FIELD}}().subtract(delta));
repository.save(e); // sans verrou → lost update possible

// ❌ BLOCAGE — @Version sans gestion de l'exception
repository.save(e); // OptimisticLockException → 500 silencieuse
```

## Pattern conforme
```java
// ✅ Fetch join — une requête pour les associations
@EntityGraph(attributePaths = {"{{ASSOCIATION_PATH}}"})
@Query("SELECT e FROM {{ENTITY}} e WHERE e.{{FILTER_FIELD}} = :value")
List<{{DTO}}> findWithAssociations(@Param("value") {{FILTER_TYPE}} value, Pageable page);

// ✅ Pagination keyset — page profonde reste rapide
@Query("SELECT e FROM {{ENTITY}} e WHERE e.{{SORT_FIELD}} > :lastValue ORDER BY e.{{SORT_FIELD}} LIMIT :n")
List<{{DTO}}> findPage(@Param("lastValue") {{SORT_TYPE}} lastValue, @Param("n") int n);

// ✅ Décrément atomique — pas de lost update
@Modifying
@Query("UPDATE {{ENTITY}} e SET e.{{CRITICAL_FIELD}} = e.{{CRITICAL_FIELD}} - :delta WHERE e.id = :id AND e.{{CRITICAL_FIELD}} >= :delta")
int decrementIfSufficient(@Param("id") UUID id, @Param("delta") {{FIELD_TYPE}} delta);
// 0 = insuffisant ; 1 = succès

// ✅ @Version + gestion OptimisticLockException
@Entity class {{ENTITY}} { @Version private Long version; }
try { repository.save(entity); }
catch (OptimisticLockException e) {
    throw new ConflictException("{{CONFLICT_MESSAGE}}"); // 409
}
```

## Checklist CI/CD
- [ ] Endpoint `{{LIST_ENDPOINT}}` → log SQL → 1 requête (pas 1+N)
- [ ] `findAll()` sans `Pageable` → interdit par règle Checkstyle/ArchUnit
- [ ] Test concurrent (2 threads mettent à jour `{{CRITICAL_FIELD}}`) → 1 succès, 1 refus (0 lost update)
- [ ] Test pagination : page {{DEEP_PAGE}} → temps stable < {{PAGE_LATENCY}} ms (pas OFFSET)
