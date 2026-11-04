# Angular Modern Best Practices

> Rules for any agent (Devin, Cursor, Copilot, etc.) writing or modifying Angular code in this repo.
> Always check the installed `@angular/core` version in `package.json` first and never emit syntax the project's version does not support. Target: Angular 17 through 22 (22 is current as of June 2026). Do not rewrite unrelated code; modernize only what you touch unless asked otherwise.


Generate Angular code the way the current Angular team recommends. Legacy patterns
(structural directives, NgModules, decorator-based inputs, constructor injection) still
work, but new code should not use them, and touched code should be modernized when it is
safe to do so.

## Step 0 - Detect the project's Angular version

Before writing code, check `package.json` (`@angular/core`) or run `ng version`.
Adapt to what is installed; never emit syntax the project's version does not support.

| Feature | Min version |
|---|---|
| `@if` / `@for` / `@switch` control flow (stable) | 17 (stable in 18) |
| `@defer` blocks | 17 (stable in 18) |
| Signals, `computed`, `effect` | 16 (stable 17+; `effect` stable 20) |
| Signal `input()` / `output()` / `model()` / `viewChild()` | 17.1 - 17.3 (stable 19) |
| `@let` template variables | 18.1 (stable 19) |
| Standalone is the default (omit `standalone: true`) | 19 |
| `linkedSignal`, `resource()`, `httpResource` (experimental) | 19 |
| Zoneless change detection (`provideZonelessChangeDetection`) | 20 stable |
| Style guide: no `.component` / `.service` suffix in new files | 20 |
| Zoneless is the default for new projects; Vitest default; Signal Forms experimental | 21 |
| OnPush is the default change detection; Signal Forms and resources stable; `@Service()`; HttpClient uses Fetch by default; requires TypeScript 6, Node 22+ | 22 |

If the version is older than a feature, either use the older equivalent or propose an
`ng update` first. If unsure about a v21/v22 API shape, check angular.dev rather than
guessing - those APIs were still moving recently (especially Signal Forms).

## Core rules (apply to all new code)

### 1. Templates: built-in control flow, never structural directives

Use `@if`, `@else`, `@for`, `@empty`, `@switch`, `@defer`, `@let`. Do not write
`*ngIf`, `*ngFor`, `*ngSwitch`, `ngTemplateOutlet`-based else templates, or
`<ng-container *ngIf>` wrappers in new code.

```html
<!-- Before -->
<div *ngIf="isInitialDataLoading" class="loading-state" role="status" aria-live="polite">
  <span class="loading-spinner" aria-hidden="true"></span>
  <span>{{ 'CASHBACK.LOADING' | translate }}</span>
</div>

<!-- After -->
@if (isInitialDataLoading()) {
  <div class="loading-state" role="status" aria-live="polite">
    <span class="loading-spinner" aria-hidden="true"></span>
    <span>{{ 'CASHBACK.LOADING' | translate }}</span>
  </div>
}
```

Rules of thumb:
- `@for` **requires** `track`. Track by a stable unique id (`track item.id`); use
  `track $index` only for static lists; never track by the object itself for data that
  gets re-fetched. Always consider `@empty`.
- Use `@else if` / `@else` instead of `*ngIf="x; else tpl"` plus `<ng-template #tpl>`.
- Use `@if (user$ | async; as user)` or, better, a signal: `@if (user(); as u)`.
- Use `@let total = price() * qty();` to avoid repeating expressions, instead of
  `*ngIf="expr as alias"` tricks.
- Use `@defer (on viewport)` / `(on interaction)` / `(on idle)` with `@placeholder`,
  `@loading`, `@error` for heavy below-the-fold content.
- Use `[class.active]="isActive()"` / `[class]="..."` and `[style.width.px]="w()"`
  instead of `ngClass` / `ngStyle`.
- Self-closing tags for components: `<app-card />`.
- Remove `CommonModule`, `NgIf`, `NgFor`, `NgClass` etc. from component `imports` once
  unused. Keep pipes (`AsyncPipe`, `DatePipe`, translate pipe, ...) imported individually.
- Avoid method calls in templates unless they are signals; use `computed()` for derived
  values.

More before/after examples: see the "Template migration cheatsheet" section at the end of this file.

### 2. Components: standalone, signal-based API, OnPush

```ts
import { ChangeDetectionStrategy, Component, computed, input, output, model, inject } from '@angular/core';

@Component({
  selector: 'app-cashback-card',
  imports: [TranslatePipe],              // no NgModule; no `standalone: true` on v19+
  templateUrl: './cashback-card.html',
  changeDetection: ChangeDetectionStrategy.OnPush, // omit on v22+ (it is the default)
  host: { class: 'cashback-card', '[class.is-active]': 'active()' },
})
export class CashbackCard {
  private readonly api = inject(CashbackService);

  readonly amount = input.required<number>();
  readonly currency = input('TND');
  readonly selected = model(false);          // two-way: [(selected)]
  readonly redeemed = output<string>();

  readonly label = computed(() => `${this.amount()} ${this.currency()}`);
  readonly active = computed(() => this.selected());
}
```

- **No NgModules** in new code. Every component/directive/pipe is standalone.
- **`input()`, `input.required()`, `output()`, `model()`** instead of `@Input()`, `@Output()
  + EventEmitter`. Use `transform`/`alias` options on `input()` when needed (`booleanAttribute`,
  `numberAttribute`).
- **`viewChild()`, `viewChildren()`, `contentChild()`, `contentChildren()`** instead of
  `@ViewChild` etc.
- **`inject()`** instead of constructor injection (fields can then be `readonly`, and it
  works in functions and with inheritance).
- **`host: {}`** metadata instead of `@HostBinding` / `@HostListener`.
- **OnPush everywhere** (explicit on <=21, implicit on 22+). With signals this is almost
  free. Do not set `ChangeDetectionStrategy.Eager` unless a legacy component truly needs it.
- Prefer small components with inline `template` for tiny ones, external files for larger.
- Mark fields `readonly` and `protected` when only the template uses them.
- Use `NgOptimizedImage` (`ngSrc`) for images; provide `width`/`height` and `priority`
  for LCP images.

### 3. State and reactivity: signals first, RxJS where it fits

- Local/component and shared state: `signal()`, `computed()`, `linkedSignal()` for derived
  state that is also writable/resettable.
- Update immutably: `items.update(list => [...list, x])`, never mutate arrays/objects in
  place (OnPush + zoneless will not see it).
- `effect()` is for side effects that bridge out of signals (logging, localStorage, DOM
  APIs). Never use `effect()` to derive state - use `computed`/`linkedSignal`. Never write
  signals inside effects unless unavoidable.
- Async data: prefer `resource()` / `httpResource()` / `rxResource()` (stable in v22) for
  request-driven state; they expose `value()`, `isLoading()`, `error()`, `reload()`.
  On older versions use `toSignal(http.get(...))`.
- RxJS stays the right tool for event streams, debouncing, websockets, complex
  orchestration (`switchMap`, `combineLatest`). Convert at the edge with `toSignal()` /
  `toObservable()` from `@angular/core/rxjs-interop`.
- Never subscribe manually in components if avoidable. If you must: `takeUntilDestroyed()`
  (or `DestroyRef`) - not `Subject` + `ngOnDestroy`. Prefer `async` pipe or `toSignal`.
- Debounce a signal: `debounced()` exists in v22 but is experimental - confirm before using.
- Lifecycle: prefer `afterNextRender` / `afterEveryRender` for DOM work, constructors /
  field initializers over `ngOnInit` where possible (zoneless + signals make `ngOnInit`
  rarely necessary). Keep `ngOnInit` only if existing code relies on it.

### 4. Dependency injection and services

```ts
@Injectable({ providedIn: 'root' })   // v22+: @Service() is the shorter equivalent
export class CashbackService {
  private readonly http = inject(HttpClient);
  getAll() { return this.http.get<Cashback[]>('/api/cashback'); }
}
```

- Always `inject()`; `providedIn: 'root'` for app-wide singletons; component-level
  `providers` for scoped state.
- Use `InjectionToken` with a `factory` for config instead of module `forRoot()` patterns.
- App config in `app.config.ts` with `provide*` functions:
  `provideRouter(routes, withComponentInputBinding(), withViewTransitions())`,
  `provideHttpClient(withInterceptors([authInterceptor]), withFetch())`,
  `provideZonelessChangeDetection()` (default in new v21+ apps),
  `provideClientHydration(withIncrementalHydration(), withEventReplay())` for SSR,
  `provideBrowserGlobalErrorListeners()`.
- **Functional** guards, resolvers, and interceptors (`CanActivateFn`, `ResolveFn`,
  `HttpInterceptorFn`) - not class-based ones.

### 5. Routing

- Lazy-load every feature: `loadComponent: () => import('./x').then(m => m.X)` and
  `loadChildren: () => import('./x.routes').then(m => m.ROUTES)`.
- Enable `withComponentInputBinding()` and receive route/query params as `input()`s
  instead of injecting `ActivatedRoute` and subscribing.
- Use `Router.navigate`/`routerLink` with typed arrays; use `redirectTo` as a function
  when logic is needed; set `title` in route config.
- Use `@defer` for in-page lazy loading, route lazy-loading for page-level splitting.

### 6. Forms

- **v22+ new forms:** prefer Signal Forms (stable): a model `signal`, `form(model, schema)`,
  schema-based validators, `FormField`/`FormRoot` directives in imports. Verify exact
  imports/selectors on angular.dev since the API was renamed during v21 -> v22.
- **v21 and earlier / existing code:** typed Reactive Forms (`FormBuilder.nonNullable`,
  `FormControl<string>`). Avoid untyped forms and `any`.
- Custom controls: Signal Forms `FormValueControl` (v22) or `ControlValueAccessor` for
  reactive forms.
- Expose form state to templates with signals / `toSignal(form.valueChanges)`.
- Avoid template-driven forms for anything non-trivial.

### 7. Zoneless

- New v21+ apps are zoneless: no `zone.js` in polyfills, no `NgZone.run` hacks.
- Change detection is triggered by signals, `markForCheck`, template events, `async`
  pipe, and `ComponentRef.setInput`. Plain property mutation will **not** update the view.
- When migrating an existing app: `provideZonelessChangeDetection()`, remove `zone.js`
  from `angular.json` polyfills, convert state to signals, test thoroughly.

### 8. Performance checklist

- `@for` with a correct `track`.
- OnPush + signals; `computed()` instead of template function calls or getters.
- Lazy routes, `@defer`, `NgOptimizedImage`.
- SSR + hydration (incremental hydration with `@defer (hydrate on ...)`) for public pages.
- `HttpClient` with `withFetch()` (default from v22) and request caching via
  `withHttpTransferCache` for SSR.
- Avoid subscribing in templates multiple times to the same observable (duplicate requests);
  use one `@let` / `toSignal`.

### 9. Testing

- v21+ new projects use **Vitest** (`ng test`); older projects use Karma/Jasmine or Jest -
  match what exists.
- Test with `TestBed` + standalone `imports: [Component]`; set inputs with
  `fixture.componentRef.setInput('amount', 10)`; await `fixture.whenStable()` for zoneless.
- Use Angular Material component **harnesses** for UI interactions.
- Prefer testing behavior (DOM output) over implementation details; provide fakes via
  `{ provide: Service, useValue: fake }`.

### 10. Style and structure

- Follow the project's existing file naming. For brand-new v20+ projects the CLI no longer
  adds suffixes (`cashback-card.ts`, class `CashbackCard`); do not rename existing files
  just to follow this.
- Feature-folder structure, one component per file, small focused services.
- Strict TypeScript: no `any`, enable `strict` and `strictTemplates`.
- Accessibility: keep ARIA roles/labels, use semantic HTML, `@if` blocks must not drop
  `role="status"`/`aria-live` regions that screen readers depend on. Consider Angular
  Aria directives (stable in v22) for complex widgets.
- Keep the project's chosen libraries (ngx-translate, Material, PrimeNG...) - modernize
  the Angular patterns, don't swap libraries unprompted.

## Migrating existing code

Prefer the official schematics - they are safe and mechanical. Run them one at a time,
commit between each, build and test:

```bash
ng update @angular/core @angular/cli                      # move to latest first
ng generate @angular/core:control-flow                     # *ngIf/*ngFor/*ngSwitch -> @if/@for/@switch
ng generate @angular/core:standalone                       # NgModules -> standalone
ng generate @angular/core:inject                           # constructor DI -> inject()
ng generate @angular/core:signal-input-migration           # @Input -> input()
ng generate @angular/core:output-migration                 # @Output -> output()
ng generate @angular/core:signal-queries-migration         # @ViewChild etc. -> viewChild()
ng generate @angular/core:self-closing-tag                 # <app-x></app-x> -> <app-x />
ng generate @angular/core:route-lazy-loading               # eager routes -> loadComponent
ng generate @angular/core:cleanup-unused-imports           # remove dead imports
```

After the control-flow schematic, manually review `@for` tracking expressions (the schematic
defaults can be `track $index`), and remove leftover `CommonModule` imports.

When asked to modernize only a snippet, convert just that snippet and mention (do not
perform) the broader migrations. When asked to review code, flag legacy patterns with the
modern replacement and a one-line reason.

## Anti-patterns to flag or fix

- `*ngIf`, `*ngFor`, `*ngSwitch`, `[ngClass]`, `[ngStyle]` in new templates
- `@for` without `track`, or `track $index` on dynamic reorderable data
- New NgModules, `forRoot()` modules, class-based guards/interceptors/resolvers
- `@Input()` / `@Output()` / `EventEmitter` / `@ViewChild` in new components
- Constructor injection when `inject()` works
- Manual `.subscribe()` without teardown; `Subject` + `ngOnDestroy` unsubscribe boilerplate
- Mutating arrays/objects in place under OnPush/zoneless
- `effect()` used to compute derived state or to set other signals
- Function/getter calls in templates that do real work
- Untyped forms, `any`, `ChangeDetectorRef.detectChanges()` sprinkled to "fix" updates
- Eager-loaded feature routes; giant components with mixed concerns

---

# Template migration cheatsheet


## @if

```html
<!-- *ngIf with else -->
<div *ngIf="items.length; else empty">...</div>
<ng-template #empty><p>No items</p></ng-template>

<!-- becomes -->
@if (items().length) {
  <div>...</div>
} @else {
  <p>No items</p>
}
```

```html
<!-- *ngIf with alias -->
<div *ngIf="user$ | async as user">{{ user.name }}</div>

<!-- becomes (observable) -->
@if (user$ | async; as user) { <div>{{ user.name }}</div> }

<!-- or (signal / @let) -->
@let user = user$ | async;
@if (user) { <div>{{ user.name }}</div> }
```

```html
<!-- chained conditions -->
@if (status() === 'loading') { <app-spinner /> }
@else if (status() === 'error') { <app-error [error]="error()" /> }
@else { <app-content [data]="data()" /> }
```

## @for

```html
<!-- *ngFor -->
<li *ngFor="let item of items; let i = index; let last = last; trackBy: trackById">{{ item.name }}</li>

<!-- becomes -->
@for (item of items(); track item.id; let i = $index, last = $last) {
  <li>{{ item.name }}</li>
} @empty {
  <li>No items</li>
}
```

Implicit variables: `$index`, `$count`, `$first`, `$last`, `$even`, `$odd`.
Track rules: stable unique id > `track $index` (static lists only) > never the object for refetched data.

## @switch

```html
@switch (role()) {
  @case ('admin') { <app-admin-panel /> }
  @case ('user')  { <app-user-panel /> }
  @default        { <app-guest-panel /> }
}
```

## @defer

```html
@defer (on viewport; prefetch on idle) {
  <app-heavy-chart [data]="data()" />
} @placeholder (minimum 200ms) {
  <div class="chart-skeleton"></div>
} @loading (after 100ms; minimum 500ms) {
  <app-spinner />
} @error {
  <p>Could not load chart.</p>
}
```

Triggers: `on idle`, `on viewport`, `on interaction`, `on hover`, `on immediate`, `on timer(5s)`, `when expr`.
With SSR incremental hydration: `@defer (hydrate on viewport) { ... }`.

## ng-container / ng-template leftovers

- `<ng-container *ngIf="x">` -> `@if (x) { ... }` (no wrapper element needed at all).
- `<ng-container *ngTemplateOutlet="tpl; context: ctx">` is still valid for reusable template fragments.
- Wrapping `*ngIf` + `*ngFor` on one element (impossible before) is now just nesting `@if` and `@for`.

## Classes and styles

```html
<!-- before -->
<div [ngClass]="{ active: isActive, disabled: isDisabled }" [ngStyle]="{ 'width.px': width }">
<!-- after -->
<div [class.active]="isActive()" [class.disabled]="isDisabled()" [style.width.px]="width()">
```

## Event / two-way binding with signals

```html
<app-toggle [(checked)]="isOn" />            <!-- isOn is a WritableSignal, child uses model() -->
<input [value]="query()" (input)="query.set($any($event.target).value)" />
```
