# Angular Study Notes (v15+ / Modern Angular)

> Targets Angular 15 through 20. Emphasis on the modern stack: **standalone components**, **signals**, **new control flow**, and the current DI/routing/HTTP APIs. Class-based `NgModule` patterns are noted where they still matter for legacy code.
>
> Coming from Python: think of Angular as a strongly-typed, decorator-driven framework. Decorators (`@Component`, `@Injectable`) behave like Python decorators but are *metadata* consumed by the compiler, not runtime wrappers. TypeScript's structural typing sits somewhere between Python's duck typing and a nominal type system.

---

## 1. Mental Model & Project Anatomy

Angular compiles templates + TS classes into an optimized JS bundle. The core building blocks:

| Concept | Role | Python analogy |
|---|---|---|
| Component | UI unit: template + class + styles | A class rendering a view |
| Directive | Behavior attached to elements | A mixin/decorator on markup |
| Service | Shared logic/state, injected | A singleton module/object |
| Signal | Reactive value container | An observable cell/property |
| Pipe | Pure transform in templates | A filter function |

**Standalone project (default since v17):**

```
src/
  main.ts              # bootstrapApplication(AppComponent, appConfig)
  app/
    app.component.ts
    app.config.ts      # providers: router, http, etc.
    app.routes.ts
```

```typescript
// main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { AppComponent } from './app/app.component';
import { appConfig } from './app/app.config';

bootstrapApplication(AppComponent, appConfig)
  .catch(err => console.error(err));
```

```typescript
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideRouter } from '@angular/router';
import { provideHttpClient } from '@angular/common/http';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    provideHttpClient(),
  ],
};
```

---

## 2. Components

### Standalone component (modern default)

> **See the dedicated deep-dive:** [`angular-standalone-components.md`](./angular-standalone-components.md) — full old-vs-new comparison, why Angular recommends standalone, and migration. Quick version below.

```typescript
import { Component } from '@angular/core';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-greeting',
  standalone: true,          // implicit/default in v19+, explicit before
  imports: [CommonModule],   // import what the template uses
  template: `<h1>Hello, {{ name }}!</h1>`,
  styles: [`h1 { color: teal; }`],
})
export class GreetingComponent {
  name = 'Angular';
}
```

Key points:
- `standalone: true` removes the need for `NgModule`. Components declare their own `imports`.
- `templateUrl` / `styleUrls` for external files; inline for small components.
- Selectors: `app-x` (element), `[appX]` (attribute), `.appX` (class).

### Lifecycle hooks (execution order)

```typescript
export class DemoComponent implements OnInit, OnChanges, OnDestroy, AfterViewInit {
  ngOnChanges(changes: SimpleChanges) {} // @Input changed (before ngOnInit first time)
  ngOnInit() {}                          // once, after first inputs set
  ngDoCheck() {}                         // every change-detection run (use sparingly)
  ngAfterContentInit() {}                // projected content ready
  ngAfterContentChecked() {}
  ngAfterViewInit() {}                   // view + child views ready (DOM available)
  ngAfterViewChecked() {}
  ngOnDestroy() {}                       // cleanup: unsubscribe, clear timers
}
```

> Modern alternative to some hooks: `afterNextRender()` / `afterRender()` (v16+) for DOM-only work, and `effect()` for reactive side effects.

---

## 3. Templates & Data Binding

```html
<!-- Interpolation -->
<p>{{ user.name }}</p>

<!-- Property binding -->
<img [src]="avatarUrl" [alt]="user.name">

<!-- Attribute / class / style binding -->
<div [attr.aria-label]="label" [class.active]="isActive" [style.width.px]="w"></div>

<!-- Event binding -->
<button (click)="save()">Save</button>
<input (keyup.enter)="submit()">

<!-- Two-way binding (needs FormsModule for ngModel) -->
<input [(ngModel)]="name">
<!-- Equivalent to: [ngModel]="name" (ngModelChange)="name = $event" -->

<!-- Template reference variable -->
<input #box (input)="log(box.value)">
```

**Two-way on custom components** uses the `xxx` / `xxxChange` convention, or the modern `model()` signal (see §5).

---

## 4. Control Flow

### New built-in syntax (v17+) — preferred

```html
@if (user.isAdmin) {
  <admin-panel />
} @else if (user.isEditor) {
  <editor-panel />
} @else {
  <p>Read-only</p>
}

@for (item of items; track item.id) {
  <li>{{ item.name }}</li>
} @empty {
  <li>No items</li>
}

@switch (status) {
  @case ('loading') { <spinner /> }
  @case ('error')   { <error-box /> }
  @default          { <content /> }
}

<!-- Lazy render on demand (v17+) -->
@defer (on viewport) {
  <heavy-chart [data]="data" />
} @placeholder {
  <p>Scroll to load…</p>
} @loading (minimum 500ms) {
  <spinner />
} @error {
  <p>Failed to load.</p>
}
```

`track` is **mandatory** in `@for` — it's the diffing key (like React's `key`). Use a stable unique id; `$index` only if items have no id.

### Legacy structural directives (still valid, common in older code)

```html
<div *ngIf="show; else other">…</div>
<ng-template #other>fallback</ng-template>

<li *ngFor="let item of items; trackBy: trackById; let i = index">{{ i }}</li>

<div [ngSwitch]="status">
  <p *ngSwitchCase="'loading'">…</p>
  <p *ngSwitchDefault>…</p>
</div>
```

Migrate with: `ng generate @angular/core:control-flow`.

---

## 5. Signals (v16+ / stable in v22) — the modern reactivity model

> **See the dedicated deep-dive:** [`angular-signals.md`](./angular-signals.md) — why signals exist, full old-vs-new (BehaviorSubject/`@Input`/`ngOnChanges`/manual CD) with examples, `linkedSignal`, the Resource API, `debounced()`, RxJS interop, and Angular 22 defaults. Quick version below.

Signals are the biggest post-v14 shift. A signal wraps a value and notifies consumers on change, enabling fine-grained, zoneless-friendly updates. In **Angular 22** the Signals API, Signal Forms, and the Resource API are all stable, and `OnPush` is the default change detection strategy.

```typescript
import { signal, computed, effect } from '@angular/core';

const count = signal(0);                      // read: count(); set: count.set(5)
const doubled = computed(() => count() * 2);  // derived, memoized, read-only
effect(() => console.log('count is', count())); // side effect on change
```

Signal component I/O replaces the decorators: `input()` / `input.required()` for `@Input`, `output()` for `@Output`, `model()` for two-way. Queries: `viewChild()` / `contentChild()`. See the deep-dive for full examples and the old-way equivalents.

---

## 6. Dependency Injection

Angular's DI is a hierarchical injector tree. Providers resolve from the closest injector upward.

```typescript
@Injectable({ providedIn: 'root' })   // app-wide singleton, tree-shakable
export class UserService {
  private http = inject(HttpClient);   // modern functional injection
  getUsers() { return this.http.get<User[]>('/api/users'); }
}
```

**Two ways to inject:**

```typescript
// Constructor injection (classic)
constructor(private users: UserService) {}

// inject() function (modern, works in field initializers & functions)
private users = inject(UserService);
```

`inject()` must run in an *injection context* (constructor, field initializer, factory, or `runInInjectionContext`).

**Provider scopes & tokens:**

```typescript
// InjectionToken for non-class values
export const API_URL = new InjectionToken<string>('API_URL');
// provide: { provide: API_URL, useValue: 'https://api.example.com' }

// Provider recipes
{ provide: Logger, useClass: ConsoleLogger }
{ provide: Logger, useExisting: RootLogger }
{ provide: CONFIG, useValue: {...} }
{ provide: Service, useFactory: () => new Service(inject(Dep)) }
```

Provide at `providers` in `app.config.ts` (root), on a component (component-scoped instance), or on a route.

---

## 7. Directives

**Attribute directive** (changes appearance/behavior):

```typescript
import { Directive, ElementRef, inject, input, HostListener } from '@angular/core';

@Directive({ selector: '[appHighlight]', standalone: true })
export class HighlightDirective {
  private el = inject(ElementRef);
  color = input('yellow', { alias: 'appHighlight' });

  @HostListener('mouseenter') onEnter() { this.set(this.color()); }
  @HostListener('mouseleave') onLeave() { this.set(''); }
  private set(c: string) { this.el.nativeElement.style.background = c; }
}
```

```html
<p [appHighlight]="'lightblue'">Hover me</p>
```

- **Structural directives** (`*ngIf`, `*ngFor`) alter DOM layout via `<ng-template>`.
- **`hostDirectives`** (v15+) compose behavior without inheritance:

```typescript
@Component({ hostDirectives: [HighlightDirective], /* … */ })
```

---

## 8. Pipes

```html
{{ price | currency:'USD' }}
{{ date | date:'medium' }}
{{ obj | json }}
{{ name | uppercase }}
{{ items | slice:0:5 }}
{{ data$ | async }}          <!-- subscribes & unwraps Observable/Promise -->
```

**Custom pure pipe:**

```typescript
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({ name: 'truncate', standalone: true })
export class TruncatePipe implements PipeTransform {
  transform(value: string, limit = 20): string {
    return value.length > limit ? value.slice(0, limit) + '…' : value;
  }
}
```

Pure pipes (default) recompute only when the input reference changes — cheap and cache-friendly. Impure pipes (`pure: false`) run every CD cycle; avoid unless necessary.

---

## 9. Forms

### Reactive forms (preferred for anything non-trivial)

```typescript
import { FormBuilder, Validators, ReactiveFormsModule } from '@angular/forms';

@Component({ imports: [ReactiveFormsModule], template: `
  <form [formGroup]="form" (ngSubmit)="submit()">
    <input formControlName="email">
    @if (form.controls.email.invalid && form.controls.email.touched) {
      <small>Valid email required</small>
    }
    <button [disabled]="form.invalid">Save</button>
  </form>
`, standalone: true })
export class SignupComponent {
  private fb = inject(FormBuilder);
  form = this.fb.nonNullable.group({
    email: ['', [Validators.required, Validators.email]],
    age:   [18, [Validators.min(0)]],
  });
  submit() { if (this.form.valid) console.log(this.form.getRawValue()); }
}
```

- Typed forms (v14+): `FormBuilder`/`FormControl` infer value types; use `nonNullable` to avoid `null` in the type.
- Custom validator: a function `(control) => ValidationErrors | null`.

### Template-driven forms (simpler, small forms)

```html
<form #f="ngForm" (ngSubmit)="save(f.value)">
  <input name="email" ngModel required email>
  <button [disabled]="f.invalid">Save</button>
</form>
```
Needs `FormsModule`.

---

## 10. HTTP

```typescript
import { HttpClient, httpResource } from '@angular/common/http';

@Injectable({ providedIn: 'root' })
export class Api {
  private http = inject(HttpClient);

  getUser(id: string) {
    return this.http.get<User>(`/api/users/${id}`);   // returns Observable
  }
  create(u: User) {
    return this.http.post<User>('/api/users', u);
  }
}
```

**Functional interceptors (v15+):**

```typescript
import { HttpInterceptorFn } from '@angular/common/http';

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = inject(AuthService).token();
  return next(req.clone({ setHeaders: { Authorization: `Bearer ${token}` } }));
};

// app.config.ts
provideHttpClient(withInterceptors([authInterceptor]))
```

> `httpResource()` (v19.2+, experimental) exposes HTTP results as signals — a declarative, signal-native alternative to manual subscription.

---

## 11. Routing

```typescript
// app.routes.ts
import { Routes } from '@angular/router';

export const routes: Routes = [
  { path: '', component: HomeComponent },
  { path: 'users/:id', component: UserComponent },
  {
    path: 'admin',
    canActivate: [authGuard],
    loadComponent: () => import('./admin.component').then(m => m.AdminComponent), // lazy
  },
  { path: 'reports', loadChildren: () => import('./reports.routes').then(m => m.REPORTS) },
  { path: '**', component: NotFoundComponent },  // wildcard last
];
```

```html
<a routerLink="/users/42" routerLinkActive="active">Profile</a>
<router-outlet />
```

**Reading params (functional / signal):**

```typescript
// component input binding (withComponentInputBinding()) maps route params to @Input/input()
id = input.required<string>();   // path 'users/:id' → id populated automatically

// or imperatively
private route = inject(ActivatedRoute);
this.route.paramMap.subscribe(p => this.id = p.get('id')!);
```

**Functional guard (v15+):**

```typescript
export const authGuard: CanActivateFn = () => {
  const auth = inject(AuthService);
  const router = inject(Router);
  return auth.isLoggedIn() ? true : router.parseUrl('/login');
};
```

---

## 12. Change Detection

- Angular re-renders when it detects possible state changes. Zone.js patches async APIs (events, timers, XHR) to trigger checks.
- **OnPush** strategy limits checks to: input reference change, event from the component, or an `async` pipe emission — big perf win.

```typescript
@Component({ changeDetection: ChangeDetectionStrategy.OnPush, /* … */ })
```

- **Signals + OnPush** = fine-grained updates; the framework knows exactly which bindings depend on a changed signal.
- **Zoneless** (developer preview, v18+): `provideZonelessChangeDetection()` drops Zone.js entirely; signals/`markForCheck` drive updates. This is the direction of the framework.

---

## 13. Content Projection

```typescript
@Component({ selector: 'app-card', standalone: true, template: `
  <div class="card">
    <header><ng-content select="[card-title]"></ng-content></header>
    <div class="body"><ng-content></ng-content></div>
  </div>
`})
export class CardComponent {}
```

```html
<app-card>
  <h2 card-title>Title</h2>
  <p>Default-slot body content.</p>
</app-card>
```

`ng-content` = transclusion/slots (like passing children). `select` targets specific projected nodes.

---

## 14. RxJS Essentials (for streams & async)

Signals cover synchronous UI state; RxJS still owns event streams and complex async.

```typescript
import { debounceTime, distinctUntilChanged, switchMap, map, catchError } from 'rxjs/operators';
import { of } from 'rxjs';

results$ = this.searchControl.valueChanges.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(q => this.api.search(q).pipe(catchError(() => of([])))),
);
```

- `switchMap` cancels the previous inner request (ideal for search).
- `mergeMap` runs concurrently; `concatMap` queues; `exhaustMap` ignores new until current completes.
- Always unsubscribe: use the `async` pipe, `takeUntilDestroyed()` (v16+), or manual `Subscription`.

```typescript
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
constructor() {
  this.stream$.pipe(takeUntilDestroyed()).subscribe(/* … */);
}
```

---

## 15. Testing (brief)

```typescript
import { TestBed } from '@angular/core/testing';

describe('CounterComponent', () => {
  it('increments', () => {
    const fixture = TestBed.createComponent(CounterComponent);
    fixture.componentRef.setInput('label', 'Clicks');
    fixture.detectChanges();
    const btn = fixture.nativeElement.querySelector('button');
    btn.click();
    fixture.detectChanges();
    expect(fixture.componentInstance.value()).toBe(1);
  });
});
```

- Unit: Jasmine + Karma (default), or Jest/Vitest (v20 experimental Vitest support).
- Component harnesses (`@angular/cdk/testing`) for robust DOM interaction.
- E2E: Playwright or Cypress (Protractor is deprecated/removed).

---

## 16. CLI Cheat Sheet

```bash
ng new my-app --standalone       # scaffold (standalone is default in modern CLI)
ng serve                         # dev server
ng generate component features/user   # or: ng g c ...
ng g service core/api
ng g directive shared/highlight
ng g pipe shared/truncate
ng g guard core/auth
ng build --configuration production
ng test
ng update @angular/core @angular/cli   # version migrations w/ schematics
ng generate @angular/core:control-flow # migrate *ngIf/*ngFor → @if/@for
```

---

## 17. Version Milestones (quick reference)

| Version | Highlights |
|---|---|
| 14 | Standalone components (preview), typed reactive forms, `inject()`, `CanActivateFn` |
| 15 | Standalone stable, functional router guards, `hostDirectives`, functional interceptors |
| 16 | **Signals** (preview), `takeUntilDestroyed`, `DestroyRef`, required inputs, esbuild dev |
| 17 | New control flow (`@if/@for/@switch`), `@defer`, signals stable-ish, new docs/brand, Vite+esbuild default |
| 18 | Zoneless change detection (preview), event replay, material 3, `@angular/build` |
| 19 | Standalone `true` by default, `linkedSignal`, `resource()`, incremental hydration (preview) |
| 20 | Signal APIs stabilizing, `httpResource`, Vitest experimental, continued zoneless push |

> Exact features per minor version shift quickly. Confirm against the official changelog / update guide for your target version before relying on anything preview-flagged.

---

## 18. Coming-from-Python Gotchas

- **`this` binding:** arrow functions in class fields preserve `this`; regular methods passed as callbacks lose it. Prefer arrow fields for handlers.
- **No truthiness surprises like Python, but `0`/`''`/`NaN` are falsy** — mind `@if (count)` when `0` is valid.
- **Immutability matters with OnPush/signals:** mutating an array/object in place won't trigger updates via reference checks. Return new references (`[...arr, x]`, `{...obj}`) or use `signal.update`.
- **`async`/`await` vs Observables:** Angular APIs return Observables, not Promises. You can `firstValueFrom(obs$)` to await, but streams usually stay reactive.
- **Types are erased at runtime** (like Python hints) — DI resolves classes by reference/token, not by TS interface. You cannot inject an `interface`; use an `InjectionToken`.
- **Decorators are compile-time metadata**, unlike Python's runtime decorators — don't expect them to execute wrapping logic.
