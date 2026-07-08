# Angular Standalone Components (Deep Dive)

> Targets Angular 14+ (introduced), 15 (stable), 17 (default in `ng new`), 19 (`standalone: true` implicit). Compares the **modern standalone** approach against the **legacy `NgModule`** approach with side-by-side examples, and explains *why* Angular pushed this shift.

---

## 1. What "standalone" actually means

A standalone component/directive/pipe declares its own template dependencies directly via an `imports` array, instead of being *declared* in an `NgModule`. It becomes a self-contained, directly-importable unit.

```typescript
import { Component } from '@angular/core';
import { CommonModule } from '@angular/common';
import { UserCardComponent } from './user-card.component';

@Component({
  selector: 'app-user-list',
  standalone: true,                          // the switch
  imports: [CommonModule, UserCardComponent], // its own dependencies
  template: `
    @for (u of users; track u.id) {
      <app-user-card [user]="u" />
    }
  `,
})
export class UserListComponent {
  users = [{ id: 1, name: 'Ada' }, { id: 2, name: 'Alan' }];
}
```

> **v19+ note:** `standalone: true` is the default and can be omitted. You now write `standalone: false` only to opt a component *into* an NgModule. The examples below keep the flag explicit for clarity across versions.

---

## 2. The OLD way — NgModule-based (pre-14, still valid)

Before standalone, every component/directive/pipe had to belong to exactly one `NgModule`. The module was the unit of compilation, dependency grouping, and lazy loading.

### 2a. A feature the old way

```typescript
// user-card.component.ts
import { Component, Input } from '@angular/core';

@Component({
  selector: 'app-user-card',
  template: `<div class="card">{{ user.name }}</div>`,
})
export class UserCardComponent {
  @Input() user!: { id: number; name: string };
}
```

```typescript
// user-list.component.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-user-list',
  template: `
    <div *ngFor="let u of users; trackBy: trackById">
      <app-user-card [user]="u"></app-user-card>
    </div>
  `,
})
export class UserListComponent {
  users = [{ id: 1, name: 'Ada' }, { id: 2, name: 'Alan' }];
  trackById = (_: number, u: { id: number }) => u.id;
}
```

```typescript
// user.module.ts  ← the mandatory boilerplate
import { NgModule } from '@angular/core';
import { CommonModule } from '@angular/common';
import { UserCardComponent } from './user-card.component';
import { UserListComponent } from './user-list.component';

@NgModule({
  declarations: [UserCardComponent, UserListComponent], // components live here
  imports: [CommonModule],                              // needed for *ngFor etc.
  exports: [UserListComponent],                         // what other modules may use
})
export class UserModule {}
```

Every consumer then had to import `UserModule` (not the component) to use `UserListComponent`. Three concepts had to stay in sync: `declarations`, `imports`, `exports`.

### 2b. Old-way bootstrap

```typescript
// app.module.ts
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { HttpClientModule } from '@angular/common/http';
import { RouterModule } from '@angular/router';
import { AppComponent } from './app.component';
import { routes } from './app.routes';

@NgModule({
  declarations: [AppComponent],
  imports: [
    BrowserModule,
    HttpClientModule,
    RouterModule.forRoot(routes),
  ],
  bootstrap: [AppComponent],
})
export class AppModule {}
```

```typescript
// main.ts
import { platformBrowserDynamic } from '@angular/platform-browser-dynamic';
import { AppModule } from './app/app.module';

platformBrowserDynamic().bootstrapModule(AppModule);
```

### 2c. Old-way lazy loading

```typescript
// app-routing.module.ts
const routes: Routes = [
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.module').then(m => m.AdminModule),
  },
];
```

```typescript
// admin/admin.module.ts
@NgModule({
  declarations: [AdminComponent, AdminDashboardComponent],
  imports: [CommonModule, RouterModule.forChild(adminRoutes)],
})
export class AdminModule {}
```

You needed a whole module just to lazy-load a route.

---

## 3. The NEW way — standalone equivalents

### 3a. Same feature, standalone

```typescript
// user-card.component.ts
import { Component, input } from '@angular/core';

@Component({
  selector: 'app-user-card',
  standalone: true,
  template: `<div class="card">{{ user().name }}</div>`,
})
export class UserCardComponent {
  user = input.required<{ id: number; name: string }>();
}
```

```typescript
// user-list.component.ts
import { Component } from '@angular/core';
import { UserCardComponent } from './user-card.component';

@Component({
  selector: 'app-user-list',
  standalone: true,
  imports: [UserCardComponent],   // import the component directly — no module
  template: `
    @for (u of users; track u.id) {
      <app-user-card [user]="u" />
    }
  `,
})
export class UserListComponent {
  users = [{ id: 1, name: 'Ada' }, { id: 2, name: 'Alan' }];
}
```

No `user.module.ts`. `UserListComponent` imports exactly what its template uses. Consumers import the *component*, not a module wrapper. (Note: `@for` is built-in control flow, so `CommonModule` isn't even needed here.)

### 3b. Standalone bootstrap

```typescript
// main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { provideRouter } from '@angular/router';
import { provideHttpClient } from '@angular/common/http';
import { AppComponent } from './app/app.component';
import { routes } from './app/app.routes';

bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(routes),
    provideHttpClient(),
  ],
});
```

`BrowserModule`, `HttpClientModule`, `RouterModule.forRoot` are replaced by tree-shakable `provideX()` functions. No root `AppModule`.

### 3c. Standalone lazy loading — no module needed

```typescript
// app.routes.ts
export const routes: Routes = [
  {
    path: 'admin',
    // lazy-load a single component directly
    loadComponent: () => import('./admin/admin.component').then(m => m.AdminComponent),
  },
  {
    path: 'reports',
    // or lazy-load a set of child routes (a plain array, not a module)
    loadChildren: () => import('./reports/reports.routes').then(m => m.REPORTS_ROUTES),
  },
];
```

```typescript
// reports/reports.routes.ts
import { Routes } from '@angular/router';
export const REPORTS_ROUTES: Routes = [
  { path: '', loadComponent: () => import('./reports.component').then(m => m.ReportsComponent) },
];
```

---

## 4. Old vs New — at a glance

| Concern | NgModule (old) | Standalone (new) |
|---|---|---|
| Unit of reuse | The module | The component itself |
| Declaring deps | `declarations` + `imports` + `exports` in a module | `imports` on the component |
| Using a component | Import its `NgModule` | Import the component |
| Bootstrap | `platformBrowserDynamic().bootstrapModule(AppModule)` | `bootstrapApplication(AppComponent, config)` |
| Root providers | `AppModule.imports` (e.g. `HttpClientModule`) | `provideHttpClient()` in providers |
| Lazy load | `loadChildren` → an `NgModule` | `loadComponent` or `loadChildren` → routes array |
| Boilerplate | High (a module per feature) | Low (none) |
| Tree-shaking | Weaker (module drags along everything) | Stronger (import only what's used) |
| Testing setup | `TestBed.configureTestingModule({ declarations, imports })` | `TestBed.configureTestingModule({ imports: [Component] })` |

---

## 5. WHY Angular recommends standalone

### 5.1 Removes indirection and boilerplate
With NgModules, a component's real dependencies were split across three arrays in a *separate* file. To understand what a component needs, you had to trace `declarations`/`imports`/`exports` across modules. Standalone puts dependencies **on the component**, where they're used — one place, one source of truth.

```typescript
// You can see everything this component needs in one glance:
@Component({
  standalone: true,
  imports: [CommonModule, RouterLink, UserCardComponent, TruncatePipe],
  // ...
})
```

### 5.2 Better tree-shaking → smaller bundles
NgModules bundle things together: importing a module could pull in exports you never use, and the compiler had a harder time proving code was dead. Standalone components are individually importable, so the bundler can drop anything not referenced. Combined with `provideX()` functions (which are tree-shakable, unlike `SomethingModule.forRoot()`), unused framework features fall out of the build.

```typescript
// Only what you call is retained:
bootstrapApplication(App, { providers: [provideRouter(routes)] });
// If you never call provideAnimations(), zero animation code ships.
```

### 5.3 Simpler, cheaper lazy loading
Old lazy loading required a dedicated `NgModule` per lazy boundary. Standalone lets you lazy-load a **single component** with `loadComponent`, or a plain routes array with `loadChildren` — finer-grained code-splitting with less ceremony.

```typescript
{ path: 'chart', loadComponent: () => import('./chart').then(m => m.ChartComponent) }
```

### 5.4 Easier mental model & onboarding
The `NgModule` was a frequent source of confusion: "Why can't I use this component? Oh, it isn't exported / the module isn't imported / it's declared twice." `NgModule`-related errors (`Component X is not a known element`, `declared in multiple NgModules`) were among the most common beginner traps. Standalone collapses several concepts (module, declaration, export) into one: the component.

### 5.5 More explicit, colocated dependencies
Because imports are per-component, dependencies are precise. You import `RouterLink` only in components that route, `ReactiveFormsModule` only where you have forms. No accidental global availability leaking through a shared module, which historically hid real coupling.

### 5.6 Better tooling & future direction
Standalone is the foundation the framework is building on: signal-based inputs, `hostDirectives`, deferrable views (`@defer`), and the move toward **zoneless** all assume the standalone model. New Angular APIs are designed standalone-first. NgModules are effectively in maintenance mode — supported for legacy, not the recommended path.

### 5.7 Composition over inheritance for behavior
`hostDirectives` (standalone directives applied to a host) let you compose reusable behavior without the old pattern of shared modules or base classes.

```typescript
@Component({
  standalone: true,
  hostDirectives: [HighlightDirective, TooltipDirective],
  // ...
})
export class ButtonComponent {}
```

---

## 6. Interop: mixing old and new (migration reality)

You don't have to convert everything at once — the two models interoperate.

**Use a standalone component inside an existing NgModule:** import it in the module's `imports` (not `declarations`).

```typescript
@NgModule({
  declarations: [LegacyComponent],
  imports: [CommonModule, UserCardComponent], // standalone → goes in imports
})
export class LegacyModule {}
```

**Use an NgModule's exports inside a standalone component:** import the module in the component's `imports`.

```typescript
@Component({
  standalone: true,
  imports: [SharedModule], // an existing module works fine here
  // ...
})
export class NewComponent {}
```

**Automated migration** (v15.2+):

```bash
# 1. convert declarations to standalone
ng generate @angular/core:standalone   # choose "Convert all components…"
# 2. remove now-unnecessary NgModules
ng generate @angular/core:standalone   # choose "Remove unnecessary NgModules"
# 3. switch bootstrap to bootstrapApplication
ng generate @angular/core:standalone   # choose "Bootstrap the app using standalone APIs"
```

Run, review, test, commit between steps.

---

## 7. Common pitfalls when going standalone

- **Forgetting an import:** with modules, a shared module often provided `CommonModule`, `RouterLink`, pipes, etc. globally. Standalone components must import each dependency they actually use — expect "not a known element/pipe" errors until you add the missing import. (Built-in `@if`/`@for` need no import; `ngIf`/`ngFor`/`ngClass`/`async` still need `CommonModule`.)
- **Re-declaring the same thing:** you can't `declare` a standalone component in an NgModule; put it in `imports`.
- **Duplicated providers:** providing a service in many components (component-scoped) creates multiple instances. Use `providedIn: 'root'` or route/`ApplicationConfig` providers for singletons.
- **Barrel-file bloat undermining tree-shaking:** re-exporting everything from one `index.ts` can pull in more than intended; import from specific paths where it matters.

---

## 8. Quick reference — conversion checklist

1. Add `standalone: true` to the component/directive/pipe.
2. Move what it uses from the module's `imports`/`declarations` into the component's own `imports`.
3. Delete the now-empty feature `NgModule`; update consumers to import the component directly.
4. Replace `loadChildren → Module` with `loadComponent` or `loadChildren → routes array`.
5. Replace root modules (`HttpClientModule`, `RouterModule.forRoot`, `BrowserAnimationsModule`) with `provideHttpClient()`, `provideRouter()`, `provideAnimations()` in `bootstrapApplication`.
6. Update tests: put the component in `TestBed` `imports` instead of `declarations`.
