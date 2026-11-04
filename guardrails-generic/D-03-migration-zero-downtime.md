---
id: "D-03"
titre: "Migration zero-downtime — schéma BDD & backward compatibility"
severite: "🔴 BLOCAGE (migration qui casse la version en prod pendant le rollout)"
applicable_si: [backend]
declencheurs:
  - "Toute migration qui renomme/supprime une colonne, modifie un type ou ajoute une contrainte NOT NULL"
  - "Tout ALTER TABLE, DROP COLUMN, RENAME COLUMN dans {{MODULE_CIBLE}}"
  - "Tout déploiement en rolling update avec changement de schéma BDD"
version: "1.0"
---

## Risque
Rolling update : pendant {{ROLLOUT_DURATION}}, l'ancienne version du code et la nouvelle tournent en parallèle sur le même schéma BDD.
Un `RENAME COLUMN` / `DROP COLUMN` casse l'ancienne version instantanément → erreurs en prod avant la fin du rollout → incident.

## Règles

**Stratégie expand-contract (toujours) :**
1. **Expand** (migration V1) : ajouter la nouvelle colonne (nullable), copier les données, le nouveau code écrit dans les deux colonnes.
2. **Contract** (migration V2, sprint suivante) : une fois l'ancienne version hors prod, supprimer/renommer l'ancienne colonne.

**Règles spécifiques :**
- Jamais de `NOT NULL` sans `DEFAULT` sur une colonne existante (verrou de table sur volume important).
- `ADD COLUMN` → nullable ou avec `DEFAULT`.
- `DROP COLUMN` → uniquement après que 100 % du trafic tourne sur la nouvelle version.
- Index → `CREATE INDEX CONCURRENTLY` (pas de verrou exclusif de table).
- Versioning : `V<version>__description.sql` — jamais modifier une migration déjà appliquée.

## Anti-pattern
```sql
-- ❌ BLOCAGE — rename immédiat : casse l'ancienne version
ALTER TABLE {{TABLE}} RENAME COLUMN {{OLD_COLUMN}} TO {{NEW_COLUMN}};

-- ❌ BLOCAGE — NOT NULL sans DEFAULT (verrou long)
ALTER TABLE {{TABLE}} ADD COLUMN {{NEW_COLUMN}} {{TYPE}} NOT NULL;

-- ❌ BLOCAGE — index bloquant
CREATE INDEX idx_{{TABLE}}_{{COLUMN}} ON {{TABLE}}({{COLUMN}});
```

## Pattern conforme
```sql
-- ✅ Expand — V1 : nouvelle colonne nullable + copie
ALTER TABLE {{TABLE}} ADD COLUMN {{NEW_COLUMN}} {{TYPE}};
UPDATE {{TABLE}} SET {{NEW_COLUMN}} = {{OLD_COLUMN}};
-- Code V(n+1) écrit dans les deux colonnes

-- ✅ Contract — V2 (sprint suivante, ancienne version hors prod)
ALTER TABLE {{TABLE}} DROP COLUMN {{OLD_COLUMN}};

-- ✅ NOT NULL avec DEFAULT — pas de verrou
ALTER TABLE {{TABLE}} ADD COLUMN {{NEW_COLUMN}} {{TYPE}} NOT NULL DEFAULT {{DEFAULT_VALUE}};

-- ✅ Index non bloquant
CREATE INDEX CONCURRENTLY idx_{{TABLE}}_{{COLUMN}} ON {{TABLE}}({{COLUMN}});
```

```
# Organisation des migrations — {{MIGRATION_TOOL}} (Flyway, Liquibase, Alembic...)
migrations/
  V{{DATE}}_01__add_{{NEW_COLUMN}}_to_{{TABLE}}.sql        (expand)
  V{{DATE_NEXT_SPRINT}}_01__drop_{{OLD_COLUMN}}.sql        (contract)
```

## Checklist CI/CD
- [ ] Toute migration testée en staging avec le code de la **version précédente** (simulation rolling update)
- [ ] `grep -r "RENAME COLUMN\|DROP COLUMN" migrations/` → revue manuelle obligatoire (sprint précédente OK ?)
- [ ] `grep -r "ADD COLUMN.*NOT NULL" migrations/ | grep -v "DEFAULT"` → 0
- [ ] Index → `CREATE INDEX CONCURRENTLY` sur toutes les tables > {{MIN_TABLE_SIZE}} lignes
- [ ] Rollback testé : revenir à la version précédente sans erreur de démarrage
