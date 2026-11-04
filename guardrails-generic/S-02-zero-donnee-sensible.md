---
id: "S-02"
titre: "Zéro donnée sensible — PII, secrets métier, logs, chiffrement applicatif"
severite: "🔴 BLOCAGE absolu — jamais contournable"
applicable_si: [backend, frontend]
declencheurs:
  - "Tout code traitant {{SENSITIVE_DATA_TYPE}} (PAN, IBAN, SSN, mot de passe, clé privée)"
  - "Tout log.* sur un flux sensible dans {{MODULE_CIBLE}}"
  - "Tout champ BDD, DTO, export mentionnant une donnée confidentielle"
  - "Sérialisation JSON d'un objet sensible dans un log"
version: "1.0"
---

## Risque
Une donnée {{SENSITIVE_DATA_TYPE}} dans les logs = scope d'audit élargi à toute la plateforme de logs → coût et risque de conformité.
Un `SELECT {{SENSITIVE_FIELD}} FROM {{TABLE}}` lisible en clair = violation directe des règles de protection des données (RGPD, PCI-DSS, HIPAA selon le domaine).

## Règles

**Données sensibles ({{SENSITIVE_DATA_TYPE}}) :**
- Jamais en clair : code, BDD, logs, config, API, messages de file, tests (pas de données réelles en fixture).
- Affichage autorisé : masque ou tronqué uniquement.
- Stockage autorisé : token/référence externe ou colonne chiffrée.

**Logs :**
- Tout log sur un flux sensible → passer par un `SanitizedLogContext` (liste blanche de champs autorisés).
- Champs autorisés dans les logs : `correlationId`, identifiants de référence opaques, codes de statut, montants masqués.
- Jamais : `log.*(objetSensible)`, `.toString()` d'un objet sensible, sérialisation JSON d'un objet confidentiel.
- `DEBUG`/`TRACE` désactivés sur le flux sensible en prod.

## Anti-pattern
```java
// ❌ BLOCAGE — donnée sensible dans un log
log.info("User data: {}", userObject);            // contient des données personnelles
log.error("Payment error: {}", paymentMessage);   // contient {{SENSITIVE_DATA_TYPE}}

// ❌ BLOCAGE — donnée sensible lisible en BDD
@Column(name = "{{SENSITIVE_FIELD}}")
private String sensitiveField; // en clair
```

## Pattern conforme
```java
// ✅ Log avec liste blanche de champs autorisés
log.warn("Operation failed", SanitizedLogContext.of(
    correlationId,
    reference.getOpaqueId(),   // identifiant opaque, pas la donnée sensible
    response.getStatusCode(),
    response.getPublicErrorCode()
));

// ✅ BDD — chiffrement transparent via converter
@Convert(converter = {{ENCRYPTION_CONVERTER}}.class)
@Column(name = "{{SENSITIVE_FIELD}}_encrypted")
private String sensitiveField; // chiffré, lisible dans le code Java
```

## Checklist CI/CD
- [ ] `grep -r "log.*{{SENSITIVE_KEYWORD}}" src/` → 0 résultats
- [ ] `SELECT {{SENSITIVE_FIELD}} FROM {{TABLE}} LIMIT 1` → résultat opaque/chiffré (test d'intégration)
- [ ] `LogSanitizationTest` : capturer l'appender, asserter `doesNotContain(sensitiveValue)` sur chaque ligne
- [ ] Pas de données réelles en fixture de test
- [ ] `DEBUG`/`TRACE` désactivés sur le profil prod (vérifier la config de logging)
