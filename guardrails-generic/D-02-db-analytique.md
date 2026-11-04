---
id: "D-02"
titre: "{{ANALYTICS_DB}} — Requêtes analytiques & modélisation orientée colonnes"
severite: "🟡 ALERTE (requête orientée ligne / scan sans filtre)"
applicable_si: [backend]
declencheurs:
  - "Toute requête {{ANALYTICS_DB}} pour les rapports et analytics dans {{MODULE_CIBLE}}"
  - "Tout INSERT vers {{ANALYTICS_DB}} depuis un flux temps réel"
  - "Toute jointure ou UPDATE/DELETE sur {{ANALYTICS_DB}}"
version: "1.0"
---

## Risque
Une requête orientée ligne sur {{ANALYTICS_DB}} (UPDATE, DELETE, `SELECT *`) sur {{DATA_VOLUME}} → full scan, saturation I/O, impact sur les rapports critiques.
INSERT ligne par ligne → fragmentation → dégradation progressive des lectures.

## Règles

- {{ANALYTICS_DB}} = OLAP, lecture analytique — jamais de lecture transactionnelle (utiliser {{TRANSACTIONAL_DB}} pour ça).
- `SELECT` → toujours colonnes explicites (jamais `SELECT *`), filtrer sur les colonnes d'index/partition.
- `INSERT` → par batch (≥ `{{BATCH_SIZE}}` lignes ou ≥ `{{BATCH_INTERVAL}}`) — jamais ligne par ligne.
- Pas de `UPDATE`/`DELETE` fréquents → concevoir en append-only + versioning/partitioning.
- Partitioning : par `{{PARTITION_FIELD}}` (ex. mois, région) pour les grandes tables.

## Anti-pattern
```sql
-- ❌ ALERTE — SELECT * sur {{ANALYTICS_DB}}
SELECT * FROM {{ANALYTICS_TABLE}} WHERE {{FILTER_FIELD}} = '{{FILTER_VALUE}}';

-- ❌ ALERTE — INSERT ligne par ligne
INSERT INTO {{ANALYTICS_TABLE}} VALUES (...); -- × {{INSERT_VOLUME}} = fragmentation

-- ❌ ALERTE — UPDATE fréquent (coûteux)
UPDATE {{ANALYTICS_TABLE}} SET status = '{{NEW_STATUS}}' WHERE id = :id;
```

## Pattern conforme
```sql
-- ✅ SELECT colonnes utiles + filtre sur partition
SELECT
    {{GROUP_BY_FIELD}},
    SUM({{METRIC_FIELD}}) AS total,
    COUNT(*) AS count
FROM {{ANALYTICS_TABLE}}
WHERE {{PARTITION_FIELD}} = {{PARTITION_VALUE}}  -- filtre sur partition
  AND {{FILTER_FIELD}} = '{{FILTER_VALUE}}'
GROUP BY {{GROUP_BY_FIELD}}
ORDER BY {{GROUP_BY_FIELD}};
```

```java
// ✅ INSERT batch
List<{{EVENT_TYPE}}> batch = new ArrayList<>();
if (batch.size() >= {{BATCH_SIZE}} || elapsed > {{BATCH_INTERVAL_MS}}) {
    analyticsClient.insertBatch("{{ANALYTICS_TABLE}}", batch);
    batch.clear();
}
```

```sql
-- ✅ Append-only + versioning au lieu d'UPDATE
CREATE TABLE {{ANALYTICS_TABLE}} (
    id UUID,
    status VARCHAR,
    version INT,    -- incrémenter à chaque changement
    {{TIMESTAMP_FIELD}} TIMESTAMP
) ENGINE = {{ANALYTICS_ENGINE}}(version)   -- ex: ReplacingMergeTree, ou partition + MERGE
PARTITION BY {{PARTITION_EXPRESSION}}
ORDER BY ({{SORT_FIELDS}});
```

## Checklist CI/CD
- [ ] `grep -r "SELECT \*" src/ | grep -i "{{ANALYTICS_DB_KEYWORD}}"` → 0
- [ ] INSERT {{ANALYTICS_TABLE}} → vérifier batch size ≥ {{BATCH_SIZE}} lignes
- [ ] Pas de `UPDATE`/`DELETE` dans les requêtes analytiques planifiées
- [ ] Test : rapport sur {{DATA_VOLUME}} → < {{MAX_QUERY_TIME}} s (filtré sur partition)
