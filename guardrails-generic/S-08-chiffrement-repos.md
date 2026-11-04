---
id: "S-08"
titre: "Chiffrement au repos — colonnes BDD sensibles, fichiers, backups"
severite: "🔴 BLOCAGE (donnée sensible non chiffrée en BDD)"
applicable_si: [backend]
declencheurs:
  - "Toute nouvelle colonne BDD stockant {{SENSITIVE_DATA_TYPE}}"
  - "Toute entité avec champ potentiellement sensible ({{SENSITIVE_FIELD_EXAMPLES}})"
  - "Tout export de données (backup BDD, dump) sans chiffrement"
version: "1.0"
---

## Risque
Un dump {{DATA_STORE}} non chiffré contenant `{{SENSITIVE_FIELD}}` → données sensibles lisibles immédiatement.
Un backup {{STORAGE}} sans chiffrement = violation de {{COMPLIANCE_STANDARD}} → risque de conformité.

## Règles

- Colonnes sensibles (`{{SENSITIVE_FIELD_EXAMPLES}}`) → converter de chiffrement transparent (`@Convert`).
- Algorithme : AES-256-GCM, IV aléatoire (12 bytes) **par valeur** — jamais ECB, jamais IV fixe.
- Clé de chiffrement → gestionnaire de secrets ({{SECRET_STORE}}, rotation annuelle) — jamais dans la config applicative.
- Backup → chiffré (`gpg --encrypt`, volume chiffré, ou service de stockage avec SSE).
- Vérification : `SELECT {{SENSITIVE_FIELD}} FROM {{TABLE}}` → résultat opaque (jamais lisible).

## Anti-pattern
```java
// ❌ BLOCAGE — donnée sensible en clair en BDD
@Column(name = "{{SENSITIVE_FIELD}}")
private String sensitiveField; // lisible dans un dump

// ❌ BLOCAGE — AES-ECB (même plaintext → même ciphertext)
Cipher cipher = Cipher.getInstance("AES/ECB/PKCS5Padding");

// ❌ BLOCAGE — clé en dur
private static final byte[] KEY = "hardcoded-key!".getBytes();
```

## Pattern conforme
```java
// ✅ JPA AttributeConverter transparent — AES-256-GCM
@Converter
public class {{ENCRYPTION_CONVERTER}} implements AttributeConverter<String, String> {

    @Value("${{{ENCRYPTION_KEY_ENV_VAR}}}") private String base64Key;

    @Override public String convertToDatabaseColumn(String plaintext) {
        if (plaintext == null) return null;
        byte[] iv = new byte[12];
        new SecureRandom().nextBytes(iv);       // IV aléatoire par valeur
        Cipher c = Cipher.getInstance("AES/GCM/NoPadding");
        c.init(ENCRYPT_MODE, secretKey(), new GCMParameterSpec(128, iv));
        byte[] ct = c.doFinal(plaintext.getBytes(UTF_8));
        return Base64.getEncoder().encodeToString(concat(iv, ct));
    }

    @Override public String convertToEntityAttribute(String stored) {
        if (stored == null) return null;
        byte[] raw = Base64.getDecoder().decode(stored);
        byte[] iv  = Arrays.copyOf(raw, 12);
        byte[] ct  = Arrays.copyOfRange(raw, 12, raw.length);
        Cipher c = Cipher.getInstance("AES/GCM/NoPadding");
        c.init(DECRYPT_MODE, secretKey(), new GCMParameterSpec(128, iv));
        return new String(c.doFinal(ct), UTF_8);
    }
}

// ✅ Entité avec chiffrement transparent
@Entity public class {{ENTITY}} {
    @Convert(converter = {{ENCRYPTION_CONVERTER}}.class)
    @Column(name = "{{SENSITIVE_FIELD}}_encrypted")
    private String sensitiveField;
}
```

## Checklist CI/CD
- [ ] `SELECT {{SENSITIVE_FIELD}} FROM {{TABLE}} LIMIT 1` → résultat Base64 opaque (test d'intégration)
- [ ] Clé `{{ENCRYPTION_KEY_ENV_VAR}}` dans {{SECRET_STORE}}, absente de la config applicative
- [ ] AES/GCM/NoPadding — `grep -r "AES/ECB" src/` → 0
- [ ] IV unique par valeur : deux chiffrements du même plaintext → deux ciphertexts différents (test)
- [ ] Backup : `ls -la backups/` → tous les fichiers chiffrés (pas de dump en clair)
