---
id: "F-04"
titre: "Error Boundaries — zéro écran blanc, zéro stack exposée, format d'erreur structuré"
severite: "🔴 BLOCAGE (stack/PII à l'utilisateur) — 🟡 ALERTE (état erreur non géré)"
applicable_si: [frontend]
declencheurs:
  - "Tout composant appelant l'API dans {{MODULE_CIBLE}}"
  - "Tout affichage d'erreur à l'utilisateur"
  - "Tout état vide / chargement / intermédiaire dans l'UI"
version: "1.0"
---

## Risque
Écran blanc sur {{HTTP_ERROR_CODE}} = l'utilisateur pense que tout est perdu → appel support + potentiel double-clic.
Stack trace dans l'UI → fuite d'informations techniques → surface d'attaque.

## Règles

- Chaque composant API gère **4 états** : chargement (skeleton), succès, vide, erreur.
- Erreur HTTP → parse format structuré (`{{ERROR_FORMAT}}`) :
  - 4xx → erreur client définitive, non rejouable.
  - 5xx → erreur technique, rejouable avec backoff.
- Jamais de stack trace, code HTTP nu, message technique affiché à l'utilisateur.
- État intermédiaire → indicateur visible + message rassurant.
- Retry UI → borné (backoff + limite max {{MAX_RETRY}} tentatives — fiche B-03).
- Aucune donnée sensible ({{SENSITIVE_DATA_TYPES}}) dans le message d'erreur.

## Anti-pattern
```typescript
// ❌ BLOCAGE — stack/message technique dans l'UI
this.{{SERVICE}}.getData().subscribe({
  error: (err) => {
    this.errorMsg = err.message; // peut contenir une stack trace ou des données internes
  }
});

// ❌ ALERTE — état vide = écran blanc
@Component({ template: `<div>{{ items | async }}</div>` })
// Aucune gestion de vide, chargement, erreur
```

## Pattern conforme
```typescript
// ✅ Handler d'erreur avec format structuré ({{ERROR_FORMAT}})
@Injectable()
export class ErrorHandler {
    categorize(err: HttpErrorResponse): UserFacingError {
        const problem = err.error as {{ERROR_RESPONSE_TYPE}};
        return {
            messageKey: this.toI18nKey(problem.{{ERROR_TYPE_FIELD}}, err.status),
            retryable:  err.status >= 500, // 4xx = définitif, 5xx = rejouable
            action:     err.status >= 500 ? 'retry' : 'contact-support',
        };
    }
}

// ✅ Composant — 4 états
@Component({
  template: `
    @if (loading()) { <{{LOADING_COMPONENT}} /> }
    @else if (error()) {
      <{{ERROR_COMPONENT}}
        [message]="error()!.messageKey | translate"
        [showRetry]="error()!.retryable"
        (retry)="reload()" />
    }
    @else if (items().length === 0) {
      <{{EMPTY_STATE_COMPONENT}} [message]="'{{EMPTY_MESSAGE_KEY}}'" />
    }
    @else {
      @for (item of items(); track item.id) { <{{ITEM_COMPONENT}} [item]="item" /> }
    }
    @if (hasPending()) {
      <{{PENDING_BANNER}} [message]="'{{PENDING_MESSAGE_KEY}}'" />
    }
  `
})
export class {{LIST_COMPONENT}} {
  loading   = signal(false);
  error     = signal<UserFacingError | null>(null);
  items     = signal<{{ITEM_TYPE}}[]>([]);
  hasPending = computed(() => this.items().some(i => i.status === '{{PENDING_STATUS}}'));
}
```

## Checklist CI/CD
- [ ] Test : forcer {{HTTP_ERROR_CODE}} → message localisé + bouton Réessayer (pas d'écran blanc, pas de stack)
- [ ] Test : liste vide → `{{EMPTY_MESSAGE_KEY}}` affiché (pas d'écran blanc)
- [ ] Test : état intermédiaire → indicateur + message rassurant
- [ ] Test : réponse erreur contenant une donnée sensible → champ non affiché à l'utilisateur
- [ ] `grep -r "err.message\|error.stack" src/{{FRONTEND_SRC_PATH}}/ | grep -v "log\|console"` → 0
