---
id: "S-05"
titre: "SQL Injection — zéro concaténation dans les requêtes"
severite: "🔴 BLOCAGE"
applicable_si: [backend]
declencheurs:
  - "Toute requête SQL/JPQL/ORM construite par concaténation dans {{MODULE_CIBLE}}"
  - "Tout LIKE '%' + input + '%' ou String.format() dans une requête"
  - "Tout ORDER BY ou GROUP BY dynamique construit depuis un paramètre utilisateur"
version: "1.0"
---

## Risque
Une concaténation dans une requête SQL = injection possible → extraction de la table `{{TABLE}}`, suppression de données, escalade de privilèges dans {{MODULE_CIBLE}}.
Sur une BDD sensible ({{DATA_CLASSIFICATION}}), une injection = violation de conformité immédiate.

## Règles

- Toute valeur dynamique dans une requête → paramètre lié (`:param`, `?`, `$1`).
- `ORDER BY`/`GROUP BY` dynamiques → liste blanche d'enum, jamais concaténation de l'input.
- `LIKE` dynamique → `LIKE :pattern` avec `%` ajouté côté applicatif (`:pattern = "%" + input + "%"`).
- Aucun `String.format()`/concaténation `+` pour construire une requête.

## Anti-pattern
```java
// ❌ BLOCAGE — injection directe
String query = "SELECT * FROM {{TABLE}} WHERE user_id = '" + userId + "'";
// userId = "' OR '1'='1" → exfiltration totale

// ❌ BLOCAGE — ORDER BY injectable
String sql = "SELECT * FROM {{TABLE}} ORDER BY " + sortField;
// sortField = "1; DROP TABLE {{TABLE}}"
```

## Pattern conforme
```java
// ✅ Paramètre lié — ORM
@Query("SELECT e FROM {{ENTITY}} e WHERE e.userId = :userId AND e.status = :status")
List<{{ENTITY}}> findByUserAndStatus(
    @Param("userId") String userId,
    @Param("status") {{STATUS_ENUM}} status
);

// ✅ ORDER BY — liste blanche
public enum SortField {
    CREATED_AT("e.createdAt"), AMOUNT("e.amount");
    private final String expr;
    public static SortField fromString(String s) {
        return Arrays.stream(values())
            .filter(f -> f.name().equalsIgnoreCase(s))
            .findFirst().orElse(CREATED_AT); // jamais l'input brut
    }
}

// ✅ LIKE — % en applicatif, paramètre lié
@Query("SELECT e FROM {{ENTITY}} e WHERE e.reference LIKE :pattern")
List<{{ENTITY}}> search(@Param("pattern") String pattern);
// Appel : repo.search("%" + userInput.trim() + "%")
```

## Checklist CI/CD
- [ ] `grep -r "createQuery.*+\|createNativeQuery.*+" src/` → 0 concaténation dans requête
- [ ] Sonar S2077 → 0 issue ouverte ou "Safe" avec commentaire
- [ ] Test d'injection : `userId = "' OR '1'='1"` → `EmptyResultDataAccessException`, jamais toutes les lignes
