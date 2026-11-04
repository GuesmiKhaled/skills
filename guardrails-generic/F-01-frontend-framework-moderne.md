---
id: "F-01"
titre: "{{FRONTEND_FRAMEWORK}} — Composants modernes & gestion d'état réactive"
severite: "🟡 ALERTE (patterns deprecated sur code nouveau)"
applicable_si: [frontend]
declencheurs:
  - "Création d'un nouveau composant {{FRONTEND_FRAMEWORK}} dans {{MODULE_CIBLE}}"
  - "Utilisation de patterns dépréciés ({{DEPRECATED_PATTERNS}}) sur code nouveau"
  - "Gestion d'état par mutation directe au lieu de patterns réactifs"
version: "1.0"
---

## Risque
Composants avec des patterns dépréciés de {{FRONTEND_FRAMEWORK}} dans {{MODULE_CIBLE}} : dépendances qui s'accumulent, détection de changement lente → latence UI perceptible sur les écrans avec beaucoup de données.

## Règles

- Tout nouveau composant → patterns recommandés par la version {{FRAMEWORK_VERSION}} de {{FRONTEND_FRAMEWORK}}.
- Gestion d'état local → mécanisme réactif natif (`{{REACTIVE_STATE_MECHANISM}}`).
- Templates → syntaxe moderne (`{{MODERN_TEMPLATE_SYNTAX}}`).
- Composants sans état externe → stratégie de détection de changement optimisée (`{{CHANGE_DETECTION_STRATEGY}}`).

## Anti-pattern
```typescript
// ❌ ALERTE — patterns deprecated sur code nouveau dans {{MODULE_CIBLE}}
// {{DEPRECATED_PATTERN_EXAMPLE}}

// Gestion d'état par Subject/BehaviorSubject quand un Signal suffit
import { BehaviorSubject } from 'rxjs';
loading$ = new BehaviorSubject<boolean>(false);

// Syntaxe deprecated
// *ngIf="condition" au lieu de @if (condition)
// *ngFor="let item of items" au lieu de @for (item of items; track item.id)
```

## Pattern conforme
```typescript
// ✅ Composant avec patterns modernes {{FRONTEND_FRAMEWORK}} {{FRAMEWORK_VERSION}}
@Component({
  selector: 'app-{{COMPONENT_NAME}}',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    @if (loading()) {
      <{{LOADING_COMPONENT}} />
    } @else if (error()) {
      <{{ERROR_COMPONENT}} [message]="error()" />
    } @else {
      @for (item of items(); track item.id) {
        <{{ITEM_COMPONENT}} [item]="item" />
      }
    }
  `
})
export class {{COMPONENT_CLASS_NAME}} {
  loading = signal(false);
  error   = signal<string | null>(null);
  items   = signal<{{ITEM_TYPE}}[]>([]);
  count   = computed(() => this.items().length);
}
```

## Checklist CI/CD
- [ ] `grep -r "{{DEPRECATED_PATTERN_GREP}}" src/{{FRONTEND_SRC_PATH}}/` → 0 sur nouveaux composants
- [ ] Tout nouveau composant a `standalone: true` et `{{CHANGE_DETECTION_STRATEGY}}`
- [ ] Lint règle : pas de patterns deprecated dans les nouveaux fichiers (ESLint rule)
