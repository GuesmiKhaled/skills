---
id: "B-01"
titre: "Money Safety — BigDecimal, comparaison & arrondis"
severite: "🔴 BLOCAGE absolu"
applicable_si: [backend]
declencheurs:
  - "Tout calcul de montant dans {{MODULE_CIBLE}} : fee, total, solde, remboursement"
  - "Champs float/double/int portant un montant financier"
  - "Comparaison de montants avec == ou .equals() sans compareTo"
version: "1.0"
---

## Risque
`float` + `double` sont des approximations binaires : `0.1 + 0.2 ≠ 0.3` en virgule flottante IEEE 754.
Sur un montant de {{TRANSACTION_VOLUME}} × {{UNIT_AMOUNT}} → écart comptable cumulé non justifiable à l'audit.

## Règles

- Tout montant financier dans {{MODULE_CIBLE}} → `BigDecimal` obligatoire.
- `float`/`double`/`int`/`long` sur un montant → BLOCAGE immédiat.
- Arrondi → `RoundingMode` explicite toujours précisé.
  - Règle générale : `HALF_UP` ; Calculs fiscaux : `HALF_EVEN`.
- Comparaison → `compareTo()` uniquement — jamais `==` ni `.equals()` sur BigDecimal.
- Constructeur → `new BigDecimal("10.50")` (String) — jamais `new BigDecimal(10.50)` (double approximé).
- DTO/Command → `BigDecimal` avec `@DecimalMin("0.01")` (fiche S-06).

## Anti-pattern
```java
// ❌ BLOCAGE — float sur un montant
double total = 0.1 + 0.2; // = 0.30000000000000004

// ❌ BLOCAGE — constructeur double
BigDecimal amount = new BigDecimal(10.50); // ≠ 10.50 exact

// ❌ BLOCAGE — .equals() sur BigDecimal
if (refund.equals(originalAmount)) { ... }
// BigDecimal("10.50").equals(BigDecimal("10.5")) = false !

// ❌ BLOCAGE — arrondi implicite
BigDecimal fee = amount.multiply(rate); // arrondi indéfini
```

## Pattern conforme
```java
// ✅ BigDecimal depuis String
BigDecimal amount = new BigDecimal("{{AMOUNT_VALUE}}");

// ✅ Calcul avec RoundingMode explicite
BigDecimal fee = amount
    .multiply(feeRate)
    .setScale(2, RoundingMode.HALF_UP);

// ✅ Comparaison avec compareTo
if (refund.compareTo(originalAmount) > 0) {
    throw new BusinessException("{{REFUND_EXCEEDS_ORIGINAL_MESSAGE}}");
}

// ✅ DTO avec contrainte
record {{COMMAND}}(
    @NotNull @DecimalMin("0.01") BigDecimal amount,
    @NotBlank String currency
) {}
```

## Checklist CI/CD
- [ ] `grep -r "float\|double" src/main/ | grep -i "amount\|fee\|balance\|total"` → 0
- [ ] `grep -r "new BigDecimal([0-9]" src/` → 0 (constructeur double interdit)
- [ ] `grep -r "\.equals(" src/ | grep -i "bigdecimal\|amount\|fee"` → 0
- [ ] Test mutation : comparaison `> 0` → mutant `>= 0` tué par le test de bordure exacte
