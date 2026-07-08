# Angular Signals (Deep Dive)

> Signals were introduced in **v16** (preview), stabilized progressively, and in **Angular 22 (released June 3, 2026)** the whole signal ecosystem — the Signals API, **Signal Forms**, and the **Resource API** (`resource` / `rxResource` / `httpResource`) — is stable and production-ready. Angular 22 also makes **OnPush the default change detection strategy** and pushes **zoneless** as the default direction, both of which are designed around signals.
>
> **Official docs:** https://angular.dev/guide/signals (and the reactivity, resource, and signal-forms guides linked from there). Always confirm preview-flagged bits against angular.dev for your exact version.
>
> Coming from Python: a signal is like a reactive cell — a value plus an observer list. Reading it inside a reactive context auto-subscribes; writing it notifies. Think "spreadsheet cell" more than "event emitter."

---

## 1. Why signals exist — the need

### The problem before signals
Angular historically detected changes with **Zone.js**, which monkey-patches every async API (events, `setTimeout`, XHR, promises). When *anything* async fired, Angular re-checked **large parts of the component tree** to see what might have changed. This worked but had costs:

- **Coarse-grained.** A click anywhere could trigger checking many unrelated components.
- **Opaque.** You couldn't easily see *what* depended on *what*; performance tuning meant reaching for `OnPush`, `ChangeDetectorRef`, `trackBy`, and `NgZone.runOutsideAngular`.
- **RxJS overhead for simple state.** Local UI state (a counter, a toggle, a derived label) often got modeled with `BehaviorSubject` + `async` pipe or manual subscriptions — powerful but heavy for trivial values, and easy to leak if you forgot to unsubscribe.
- **Bundle weight.** Zone.js itself ships bytes and adds runtime cost.

### What signals fix
- **Fine-grained reactivity.** The framework knows *exactly* which template bindings read which signals, so only those update — no full-tree sweeps.
- **Explicit dependency graph.** `computed`/`effect` track their reads automatically; the data flow is inspectable (Angular DevTools shows a signal graph).
- **Less RxJS for local state.** Synchronous UI state becomes trivial and leak-free (no subscription to clean up).
- **Zoneless-ready.** With signals driving updates, Angular can drop Zone.js entirely for smaller, faster apps — the default direction in v22.
- **Ergonomic derived state.** `computed` gives memoized derivations for free.

> Nuance (from the v22 ecosystem): **signals do not replace RxJS.** RxJS still owns streams, event composition, and complex async orchestration. Signals own synchronous UI state and derived values. They interoperate.

---

## 2. Core primitives — `signal`, `computed`, `effect`

```typescript
import { signal, computed, effect } from '@angular/core';

// Writable signal
const count = signal(0);          // WritableSignal<number>
count();                          // read  → 0
count.set(5);                     // set   → 5
count.update(n => n + 1);         // derive from previous → 6

// Computed (derived, memoized, read-only)
const doubled = computed(() => count() * 2);
doubled();                        // 12 — recomputes only when `count` changes

// Effect (side effects; re-runs when its signal deps change)
effect(() => {
  console.log('count is now', count());
});
```

Rules:
- **`computed` must be pure** — no side effects, no writes. It's lazy and memoized.
- **`effect` is for side effects** — logging, syncing to `localStorage`, bridging to non-signal code. It runs in an *injection context* (constructor / field initializer) and auto-cleans on destroy.
- Reads inside `computed`/`effect`/templates are **auto-tracked** as dependencies. Reads elsewhere are not.
- Writing a signal inside an `effect` that also reads it risks loops — avoid; use `computed` for derivation.

### Equality & object signals
By default signals use `Object.is` equality. Mutating an object/array in place won't notify (same reference). Set a **new reference**, or provide a custom `equal`:

```typescript
const user = signal({ name: 'Ada', age: 36 });
user.update(u => ({ ...u, age: u.age + 1 }));   // new reference → notifies

const list = signal<number[]>([]);
list.update(xs => [...xs, 42]);                 // not xs.push(42)

const p = signal({ x: 0 }, { equal: (a, b) => a.x === b.x });
```

---

## 3. Signal-based component I/O (`input` / `output` / `model`)

The signal versions replace `@Input()` / `@Output()` / `EventEmitter`.

```typescript
import { Component, input, output, model, computed } from '@angular/core';

@Component({
  selector: 'app-counter',
  standalone: true,
  template: `
    <p>{{ label() }}: {{ value() }}</p>
    <button (click)="value.set(value() + step())">+{{ step() }}</button>
    <button (click)="reset.emit()">reset</button>
    <small>{{ display() }}</small>
  `,
})
export class CounterComponent {
  label = input.required<string>();   // required input, read as label()
  step  = input(1);                   // optional input, default 1
  value = model(0);                   // two-way: <app-counter [(value)]="n" />
  reset = output<void>();             // replaces @Output + EventEmitter
  display = computed(() => `${this.label()} = ${this.value()}`);
}
```

Notes:
- `input()` signals are **read-only** from inside the component (the parent owns them).
- `model()` is writable and two-way bindable: parent writes via `[(value)]`, child writes via `value.set(...)`.
- Alias/transform options: `input(0, { alias: 'count', transform: numberAttribute })`.

### Signal queries (replace `@ViewChild` / `@ContentChild`)

```typescript
import { viewChild, viewChildren, contentChild } from '@angular/core';

box   = viewChild<ElementRef>('box');       // Signal<ElementRef | undefined>
items = viewChildren(ItemComponent);        // Signal<readonly ItemComponent[]>
header = contentChild(HeaderComponent);
required = viewChild.required('box');        // Signal<ElementRef>
```

---

## 4. `linkedSignal` — writable derived state (stable in v22)

A `computed` is read-only. Sometimes you want derived state that also stays **locally writable** (e.g. a selected item that resets when the source list changes but can be overridden by the user). That's `linkedSignal`.

```typescript
import { signal, linkedSignal } from '@angular/core';

const options = signal(['S', 'M', 'L']);

// derives from options, but is writable
const choice = linkedSignal(() => options()[0]);
choice();            // 'S'
choice.set('L');     // user overrides → 'L'
options.set(['XS', 'XL']); // source changes → resets to 'XS'

// advanced form: react to previous value (v22 supports a custom `set`/equal)
const selected = linkedSignal({
  source: options,
  computation: (opts, prev) =>
    opts.includes(prev?.value as string) ? prev!.value : opts[0],
});
```

Use it for "derived but user-overridable" state — a very common pattern that used to require an `effect` + manual signal juggling.

---

## 5. `resource` / `rxResource` / `httpResource` — async as signals (stable in v22)

The Resource API is the signal-native way to load async data. The request re-runs automatically when a signal it reads changes, and it exposes `value()`, `status()`, `error()`, and `isLoading()` as signals.

```typescript
import { resource, signal } from '@angular/core';

const userId = signal(1);

const userResource = resource({
  params: () => ({ id: userId() }),           // reactive: re-fetches when userId changes
  loader: async ({ params, abortSignal }) => {
    const res = await fetch(`/api/users/${params.id}`, { signal: abortSignal });
    return res.json() as Promise<User>;
  },
});

// In a template:
// @if (userResource.isLoading()) { <spinner/> }
// @else if (userResource.error()) { <error/> }
// @else { {{ userResource.value()?.name }} }

userId.set(2);            // triggers a new load, previous request aborted
userResource.reload();    // manual refresh
```

**`httpResource`** — the most convenient form, built on `HttpClient`, returns a signal resource directly:

```typescript
import { httpResource } from '@angular/common/http';

const id = signal(1);
const user = httpResource<User>(() => `/api/users/${id()}`);
// user.value(), user.isLoading(), user.error() — all signals
```

**`rxResource`** bridges an RxJS-based loader into the same resource shape (useful when your data layer returns Observables).

> v22 adds an `id` option that caches the resolved value in **TransferState** during SSR, so the client skips a re-fetch after hydration.

---

## 6. RxJS interop

Signals and RxJS bridge cleanly via `@angular/core/rxjs-interop`.

```typescript
import { toSignal, toObservable } from '@angular/core/rxjs-interop';

// Observable → Signal (auto-subscribes, auto-unsubscribes)
count$ : Observable<number>;
count = toSignal(this.count$, { initialValue: 0 });

// Signal → Observable
query = signal('');
query$ = toObservable(this.query);   // emits on each change
results$ = this.query$.pipe(
  debounceTime(300),
  switchMap(q => this.api.search(q)),
);
```

Guideline: **signals for synchronous UI state and derivations; RxJS for streams and complex async.** Convert at the boundary.

### `debounced()` (new in v22)
For the classic search-box case without dropping into RxJS, v22 adds a native `debounced` signal utility that delays propagation:

```typescript
// pseudo-usage — delays downstream reactions until input settles
const term = signal('');
const debouncedTerm = debounced(term, 300);
const results = httpResource<Item[]>(() => `/api/search?q=${debouncedTerm()}`);
```

---

## 7. Signals + change detection (why v22 flips defaults)

- With signals, Angular knows precisely which bindings depend on a changed value, enabling **fine-grained updates** instead of tree-wide checks.
- **Angular 22 makes `OnPush` the default** change detection strategy for new components — signals make this the natural fit. (`ng update` sets existing components to preserve prior behavior, so upgrades don't silently change semantics.)
- Signals are the foundation of **zoneless** Angular: no Zone.js, updates driven by signal notifications — smaller bundles, faster, more predictable.

```typescript
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush,  // default in v22
  // signal reads in the template drive precise updates
})
```

---

## 8. THE OLD WAY — pre-signals, with examples

Everything below still works and appears throughout existing codebases. Signals are the *recommended* replacement, not a hard requirement.

### 8a. Local reactive state → `BehaviorSubject` + `async` pipe

```typescript
// OLD
import { Component } from '@angular/core';
import { BehaviorSubject, map } from 'rxjs';

@Component({
  selector: 'app-counter',
  template: `
    <p>Count: {{ count$ | async }}</p>
    <p>Doubled: {{ doubled$ | async }}</p>
    <button (click)="increment()">+</button>
  `,
})
export class OldCounterComponent {
  private count = new BehaviorSubject(0);
  count$ = this.count.asObservable();
  doubled$ = this.count$.pipe(map(c => c * 2));  // derived stream

  increment() { this.count.next(this.count.value + 1); }
}
```

```typescript
// NEW — signals
@Component({
  selector: 'app-counter',
  template: `
    <p>Count: {{ count() }}</p>
    <p>Doubled: {{ doubled() }}</p>
    <button (click)="increment()">+</button>
  `,
})
export class NewCounterComponent {
  count = signal(0);
  doubled = computed(() => this.count() * 2);
  increment() { this.count.update(c => c + 1); }
}
```

Fewer moving parts: no `Subject`, no `.asObservable()`, no `| async`, no derived pipe, and nothing to unsubscribe.

### 8b. Component I/O → `@Input` / `@Output`

```typescript
// OLD
import { Component, Input, Output, EventEmitter } from '@angular/core';

@Component({ selector: 'app-toggle', template: `
  <button (click)="toggle()">{{ label }}: {{ on }}</button>
`})
export class OldToggle {
  @Input() label = '';
  @Input() on = false;
  @Output() onChange = new EventEmitter<boolean>();   // for [(on)] two-way
  toggle() { this.on = !this.on; this.onChange.emit(this.on); }
}
```

```typescript
// NEW — signal inputs/model
@Component({ selector: 'app-toggle', template: `
  <button (click)="on.set(!on())">{{ label() }}: {{ on() }}</button>
`})
export class NewToggle {
  label = input('');
  on = model(false);   // [(on)] works automatically, no manual emit
}
```

### 8c. Reacting to input changes → `ngOnChanges`

```typescript
// OLD — string-keyed, untyped, verbose
export class OldChart implements OnChanges {
  @Input() data!: number[];
  total = 0;
  ngOnChanges(changes: SimpleChanges) {
    if (changes['data']) {
      this.total = this.data.reduce((a, b) => a + b, 0);
    }
  }
}
```

```typescript
// NEW — computed off a signal input; no lifecycle hook
export class NewChart {
  data = input<number[]>([]);
  total = computed(() => this.data().reduce((a, b) => a + b, 0));
}
```

### 8d. Async data → manual subscribe + `ngOnDestroy`

```typescript
// OLD — subscribe, store, unsubscribe (leak-prone)
export class OldUser implements OnInit, OnDestroy {
  user?: User;
  loading = true;
  private sub?: Subscription;
  constructor(private api: Api) {}
  ngOnInit() {
    this.sub = this.api.getUser(1).subscribe(u => {
      this.user = u; this.loading = false;
    });
  }
  ngOnDestroy() { this.sub?.unsubscribe(); }
}
```

```typescript
// NEW — resource(): reactive, auto-abort, status signals, no cleanup
export class NewUser {
  private id = signal(1);
  user = httpResource<User>(() => `/api/users/${this.id()}`);
  // template: @if (user.isLoading()) {...} @else { {{ user.value()?.name }} }
}
```

### 8e. Manual change detection → `ChangeDetectorRef`

```typescript
// OLD — poke Angular after mutating state outside its awareness
constructor(private cdr: ChangeDetectorRef) {}
onExternalEvent(v: number) {
  this.value = v;
  this.cdr.markForCheck();   // or detectChanges()
}
```

```typescript
// NEW — set a signal; the framework updates exactly what depends on it
value = signal(0);
onExternalEvent(v: number) { this.value.set(v); }
```

---

## 9. Old vs New — quick mapping

| Need | Old way | Signal way (v16+ / stable in v22) |
|---|---|---|
| Local reactive value | `BehaviorSubject` + `async` | `signal()` |
| Derived value | `.pipe(map(...))` stream | `computed()` |
| Side effect on change | `subscribe(...)` (+ unsubscribe) | `effect()` |
| Input | `@Input()` | `input()` / `input.required()` |
| Two-way input | `@Input` + `@Output xChange` | `model()` |
| Output | `@Output` + `EventEmitter` | `output()` |
| React to input change | `ngOnChanges` | `computed()` off the input signal |
| View/content query | `@ViewChild` / `@ContentChild` | `viewChild()` / `contentChild()` |
| Async data load | subscribe + loading flags + cleanup | `resource()` / `httpResource()` |
| Derived-but-writable | `effect` + extra signal | `linkedSignal()` |
| Force re-render | `ChangeDetectorRef.markForCheck()` | just `signal.set(...)` |
| Debounced input | RxJS `debounceTime` | `debounced()` (v22) |

---

## 10. Gotchas (especially coming from Python/RxJS)

- **Signals are getter functions:** you must *call* them — `count()`, not `count`. Forgetting the `()` in a template silently renders the function object.
- **Mutation ≠ notification:** `arr.push(x)` on a signal's array won't update anything. Return a new reference (`[...arr, x]`) or use `update`.
- **No side effects in `computed`:** keep it pure; put effects in `effect()`.
- **`effect` needs an injection context:** create it in a constructor/field initializer, or pass an `Injector`.
- **Don't overuse `effect` to sync signals:** if B derives from A, use `computed`, not an `effect` that writes B.
- **`toSignal` needs an initial value** (or produces `T | undefined`) — decide which you want.
- **Signals ≠ RxJS replacement:** reach for RxJS when you genuinely have a *stream* (websockets, complex event coordination, cancellation chains).

---

## 11. Migration notes

- Adopt incrementally — signals interoperate with existing `@Input`/`@Output`, RxJS, and NgModule code. No big-bang rewrite needed.
- CLI migrations exist for signal inputs/queries/outputs (e.g. `ng generate @angular/core:signal-input-migration`, `signal-queries-migration`, `output-migration`). Check `ng g @angular/core:` for the current list on your version.
- On upgrading to v22, `ng update` preserves existing components' change-detection behavior even though `OnPush` becomes the new default — so behavior won't silently change; adopt `OnPush` + signals deliberately, component by component.
- **Get to a supported version first, then adopt signals** — mixing a large version jump with a signal rewrite in one pass is a common cause of migration pain.
