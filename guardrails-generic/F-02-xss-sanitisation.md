---
id: "F-02"
titre: "XSS — Sanitisation du contenu dynamique"
severite: "🔴 BLOCAGE (HTML non sanitisé injecté dans le DOM)"
applicable_si: [frontend]
declencheurs:
  - "Tout `[innerHTML]`, `outerHTML`, `dangerouslySetInnerHTML` dans {{MODULE_CIBLE}}"
  - "Tout bypass de sanitisation (`bypassSecurityTrustHtml`, `__dangerouslySetHTML`)"
  - "Tout affichage de contenu riche issu de l'API ou d'un input utilisateur"
version: "1.0"
---

## Risque
Injection d'HTML non sanitisé → XSS : script malveillant exécuté dans le navigateur de l'utilisateur → vol de token de session, exfiltration de données, fraude.

## Règles

- `[innerHTML]` / `dangerouslySetInnerHTML` → sanitiser via `{{SANITIZER}}` ou ne jamais l'utiliser pour des données dynamiques.
- Bypass de sanitisation → BLOCAGE sauf contenu HTML interne statique documenté.
- Données utilisateur → interpolation automatique du framework (`{{ valeur }}` Angular, JSX texte React) — le framework encode automatiquement.
- Contenu riche externe → librairie `DOMPurify` + liste blanche de tags autorisés.

## Anti-pattern
```typescript
// ❌ BLOCAGE — XSS direct
// Angular
template: `<div [innerHTML]="contentFromApi"></div>` // contentFromApi non sanitisé

// React
<div dangerouslySetInnerHTML={{ __html: userContent }} /> // userContent non sanitisé

// ❌ BLOCAGE — bypass sans sanitisation
const unsafe = this.sanitizer.bypassSecurityTrustHtml(userInput);
```

## Pattern conforme
```typescript
// ✅ Interpolation — le framework sanitise automatiquement
// Angular
template: `<p>{{ contentFromApi }}</p>`

// React
<p>{contentFromApi}</p>

// ✅ Contenu HTML riche — DOMPurify avant rendu
import DOMPurify from 'dompurify';

const safeHtml = DOMPurify.sanitize(rawHtml, {
    ALLOWED_TAGS: ['b', 'i', 'p', 'br', 'strong', 'em'],
    ALLOWED_ATTR: []  // aucun attribut (pas d'onerror, href, etc.)
});

// Angular : bypass seulement après purge
const trusted = this.sanitizer.bypassSecurityTrustHtml(safeHtml);

// React : uniquement après DOMPurify
<div dangerouslySetInnerHTML={{ __html: safeHtml }} />
```

## Checklist CI/CD
- [ ] `grep -r "innerHTML\|dangerouslySetInnerHTML\|outerHTML" src/{{FRONTEND_SRC_PATH}}/` → chaque occurrence documentée et sanitisée
- [ ] `grep -r "bypassSecurityTrustHtml\|bypassSecurityTrust" src/` → 0 sur valeurs issues de l'API
- [ ] Test : payload `<img src=x onerror=alert(1)>` dans un champ → encodé, pas exécuté dans le DOM
