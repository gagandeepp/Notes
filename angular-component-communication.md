# Angular Component Communication (with Examples)

> Covers every direction of data flow between components: **parent → child**, **child → parent**, **sibling ↔ sibling**, plus **unrelated/distant** components. Uses modern Angular (v16+, signal APIs `input()`/`output()`/`model()`) as the primary approach, with the classic `@Input()`/`@Output()` equivalents shown alongside since they're still everywhere.
>
> **Quick decision guide:**
> - Parent → Child: **`input()`** (or `@Input()`)
> - Child → Parent: **`output()`** (or `@Output()` + `EventEmitter`)
> - Two-way: **`model()`** (or `[(banana-in-a-box)]`)
> - Parent reaching into child instance: **`viewChild()`** / template ref
> - Child projecting into parent shell: **content projection** (`ng-content`)
> - Siblings / distant / unrelated: **shared service** (signal or `Subject`)
> - Anything global/app-wide: **shared service** or route/state

---

## 1. Parent → Child: `input()` / `@Input()`

The parent passes data **down** by binding to the child's inputs.

### Modern (signal inputs, v17.1+)
```typescript
// child.component.ts
import { Component, input } from '@angular/core';

@Component({
  selector: 'app-user-card',
  standalone: true,
  template: `
    <div class="card">
      <h3>{{ name() }}</h3>
      <p>{{ role() }}</p>
    </div>
  `,
})
export class UserCardComponent {
  name = input.required<string>();   // required input
  role = input('Member');            // optional input with default
}
```

```typescript
// parent.component.ts
import { Component } from '@angular/core';
import { UserCardComponent } from './user-card.component';

@Component({
  selector: 'app-parent',
  standalone: true,
  imports: [UserCardComponent],
  template: `
    <app-user-card [name]="user.name" [role]="user.role" />
    <app-user-card name="Static string works too" />
  `,
})
export class ParentComponent {
  user = { name: 'Ada Lovelace', role: 'Admin' };
}
```

### Classic (`@Input()`)
```typescript
export class UserCardComponent {
  @Input() name = '';
  @Input({ required: true }) role!: string;   // required since v16
}
// parent template: <app-user-card [name]="user.name" [role]="user.role"></app-user-card>
```

### Reacting to input changes
```typescript
// Modern: derive with computed — no lifecycle hook
import { input, computed } from '@angular/core';
data = input<number[]>([]);
total = computed(() => this.data().reduce((a, b) => a + b, 0));

// Classic: ngOnChanges
@Input() data!: number[];
ngOnChanges(changes: SimpleChanges) {
  if (changes['data']) { this.total = this.data.reduce((a, b) => a + b, 0); }
}
```

### Input transforms & aliases
```typescript
import { input, numberAttribute, booleanAttribute } from '@angular/core';
count    = input(0, { transform: numberAttribute });     // "5" → 5
disabled = input(false, { transform: booleanAttribute }); // presence → true
value    = input(0, { alias: 'appValue' });               // bound as [appValue]
```

---

## 2. Child → Parent: `output()` / `@Output()`

The child notifies the parent **up** by emitting events. Data flows down via inputs; events flow up via outputs.

### Modern (signal outputs, v17.3+)
```typescript
// child.component.ts
import { Component, input, output } from '@angular/core';

@Component({
  selector: 'app-todo-item',
  standalone: true,
  template: `
    <li>
      {{ title() }}
      <button (click)="toggle.emit(id())">✓</button>
      <button (click)="remove.emit(id())">✕</button>
    </li>
  `,
})
export class TodoItemComponent {
  id = input.required<number>();
  title = input.required<string>();
  toggle = output<number>();   // replaces @Output + EventEmitter
  remove = output<number>();
}
```

```typescript
// parent.component.ts
@Component({
  selector: 'app-todo-list',
  standalone: true,
  imports: [TodoItemComponent],
  template: `
    <ul>
      @for (t of todos; track t.id) {
        <app-todo-item
          [id]="t.id" [title]="t.title"
          (toggle)="onToggle($event)"
          (remove)="onRemove($event)" />
      }
    </ul>
  `,
})
export class TodoListComponent {
  todos = [{ id: 1, title: 'Learn signals' }, { id: 2, title: 'Ship feature' }];
  onToggle(id: number) { /* $event is the emitted id */ }
  onRemove(id: number) { this.todos = this.todos.filter(t => t.id !== id); }
}
```

### Classic (`@Output()` + `EventEmitter`)
```typescript
import { Output, EventEmitter } from '@angular/core';
@Output() toggle = new EventEmitter<number>();
// this.toggle.emit(this.id);
// parent template: (toggle)="onToggle($event)"
```

> `$event` in the parent's handler is exactly the value the child passed to `.emit(...)`.

---

## 3. Two-way binding: `model()` / `[(x)]`

When the child both receives and updates a value, use two-way binding. The `[(x)]` "banana-in-a-box" is sugar for `[x]` + `(xChange)`.

### Modern (`model()`)
```typescript
// child.component.ts
import { Component, model } from '@angular/core';

@Component({
  selector: 'app-counter',
  standalone: true,
  template: `
    <button (click)="value.set(value() - 1)">−</button>
    <span>{{ value() }}</span>
    <button (click)="value.set(value() + 1)">+</button>
  `,
})
export class CounterComponent {
  value = model(0);   // writable, two-way bindable
}
```

```typescript
// parent
@Component({
  imports: [CounterComponent],
  template: `
    <app-counter [(value)]="count" />
    <p>Parent sees: {{ count }}</p>
  `,
  standalone: true,
})
export class ParentComponent {
  count = 0;   // stays in sync both directions
}
```

### Classic two-way (the convention `model()` replaces)
```typescript
@Input() value = 0;
@Output() valueChange = new EventEmitter<number>();   // MUST be named xxxChange
update(v: number) { this.value = v; this.valueChange.emit(v); }
// parent: <app-counter [(value)]="count"></app-counter>
```

---

## 4. Parent → Child (imperative): template refs & `viewChild()`

When the parent needs to **call a method** or read a property on the child instance (not just pass data), grab the child via a query.

### Template reference variable (simplest, in template)
```typescript
@Component({
  imports: [VideoPlayerComponent],
  template: `
    <app-video-player #player [src]="url" />
    <button (click)="player.play()">Play</button>
    <button (click)="player.pause()">Pause</button>
  `,
  standalone: true,
})
export class ParentComponent {
  url = '/clip.mp4';
}
// play()/pause() are public methods on VideoPlayerComponent
```

### `viewChild()` signal query (v17.2+) — access in the class
```typescript
import { Component, viewChild, ElementRef, effect } from '@angular/core';
import { VideoPlayerComponent } from './video-player.component';

@Component({
  imports: [VideoPlayerComponent],
  template: `<app-video-player [src]="url" /><input #search />`,
  standalone: true,
})
export class ParentComponent {
  player = viewChild.required(VideoPlayerComponent);   // Signal<VideoPlayerComponent>
  searchBox = viewChild<ElementRef<HTMLInputElement>>('search');

  start() { this.player().play(); }                    // call child method
  focusSearch() { this.searchBox()?.nativeElement.focus(); }
}
```

### Classic `@ViewChild`
```typescript
import { ViewChild } from '@angular/core';
@ViewChild(VideoPlayerComponent) player!: VideoPlayerComponent;
ngAfterViewInit() { this.player.play(); }   // available after view init
```

> Use imperative access sparingly — prefer inputs/outputs. Good fits: focus management, media controls, measuring DOM, integrating third-party widgets.

---

## 5. Child → Parent (structural): content projection & `contentChild()`

When a parent (a "shell"/wrapper) renders content the consumer passes *into* it, use **content projection** with `ng-content`. The parent can then query the projected children with `contentChild()`.

```typescript
// tab-group.component.ts (the shell)
import { Component, contentChildren } from '@angular/core';
import { TabComponent } from './tab.component';

@Component({
  selector: 'app-tab-group',
  standalone: true,
  template: `
    <nav>
      @for (tab of tabs(); track tab.title()) {
        <button (click)="select(tab)">{{ tab.title() }}</button>
      }
    </nav>
    <ng-content></ng-content>   <!-- projected <app-tab>s render here -->
  `,
})
export class TabGroupComponent {
  tabs = contentChildren(TabComponent);   // Signal<readonly TabComponent[]>
  select(tab: TabComponent) { this.tabs().forEach(t => t.active.set(t === tab)); }
}
```

```typescript
// usage
@Component({
  imports: [TabGroupComponent, TabComponent],
  template: `
    <app-tab-group>
      <app-tab title="Overview">…</app-tab>
      <app-tab title="Details">…</app-tab>
    </app-tab-group>
  `,
  standalone: true,
})
export class PageComponent {}
```

**Multi-slot projection** with `select`:
```typescript
template: `
  <header><ng-content select="[card-title]"></ng-content></header>
  <div class="body"><ng-content></ng-content></div>   <!-- default slot -->
`
// usage: <h2 card-title>Title</h2> goes to the header slot
```

> `@ViewChild` queries the component's **own template**; `@ContentChild`/`contentChild()` queries **projected** content. That distinction is a common interview question.

---

## 6. Sibling ↔ Sibling: shared service

Siblings have no direct binding path. Route their communication through a **shared service** they both inject. Two flavors: **signal-based** (modern, preferred for state) and **Subject-based** (RxJS, for event streams).

### Signal-based shared service (modern)
```typescript
// cart.service.ts
import { Injectable, signal, computed } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class CartService {
  private items = signal<Product[]>([]);
  readonly count = computed(() => this.items().length);
  readonly all = this.items.asReadonly();

  add(p: Product)    { this.items.update(list => [...list, p]); }
  remove(id: number) { this.items.update(list => list.filter(p => p.id !== id)); }
}
```

```typescript
// sibling A — product list, writes to the service
export class ProductListComponent {
  private cart = inject(CartService);
  addToCart(p: Product) { this.cart.add(p); }
}

// sibling B — cart badge, reads from the service (auto-updates)
@Component({ template: `🛒 {{ cart.count() }}` })
export class CartBadgeComponent {
  cart = inject(CartService);   // count() is a signal — template updates automatically
}
```

Because both siblings inject the **same singleton** (`providedIn: 'root'`), a write from A is instantly visible to B through the signal.

### Subject-based shared service (event stream)
Use when the communication is an **event/message** rather than shared state (e.g. "a notification was triggered").

```typescript
// notify.service.ts
import { Injectable } from '@angular/core';
import { Subject } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class NotifyService {
  private messages = new Subject<string>();
  readonly messages$ = this.messages.asObservable();   // read-only stream
  send(msg: string) { this.messages.next(msg); }
}
```

```typescript
// sender sibling
export class ToolbarComponent {
  private notify = inject(NotifyService);
  save() { /* … */ this.notify.send('Saved!'); }
}

// receiver sibling
export class ToastComponent {
  private notify = inject(NotifyService);
  constructor() {
    this.notify.messages$.pipe(takeUntilDestroyed()).subscribe(msg => this.show(msg));
  }
  show(msg: string) { /* display toast */ }
}
```

> **Signals vs Subject here:** use a **signal** for "current value" state that any sibling reads (cart count, selected filter). Use a **Subject** for transient **events** that should fire once per occurrence (toasts, "row clicked" broadcasts). A `BehaviorSubject` is the pre-signals equivalent of the signal store.

---

## 7. Distant / unrelated components

Same tool as siblings — a **shared service** (`providedIn: 'root'`) — because DI gives every component the same singleton regardless of tree distance. For larger apps this scales into a state-management layer:

- **Small/medium:** signal-based service (as in §6) is often enough.
- **Larger/complex:** NgRx / NgRx SignalStore, or other state libraries, for actions, selectors, effects, and devtools.
- **Cross-route data:** router (`resolve`, route params, `withComponentInputBinding()`), or a store.

```typescript
// router → component input (v16+): route params map straight to inputs
// provideRouter(routes, withComponentInputBinding())
// route: { path: 'user/:id', component: UserComponent }
export class UserComponent {
  id = input.required<string>();   // populated from the :id route param automatically
}
```

---

## 8. Special relationships

**Passing a `TemplateRef` (parent supplies markup a child renders):**
```typescript
// parent
template: `
  <app-list [rowTemplate]="row" [data]="items" />
  <ng-template #row let-item>
    <strong>{{ item.name }}</strong> — {{ item.price | currency }}
  </ng-template>
`
// child
rowTemplate = input.required<TemplateRef<any>>();
// child template: <ng-container *ngTemplateOutlet="rowTemplate(); context: { $implicit: item }" />
```

**Host → directive** (a directive reading/affecting its host) uses the same `input()`/`output()` plus `HostListener`/`host` bindings — see the directives section of the main notes.

---

## 9. Summary — pick the right tool

| Direction | Preferred (modern) | Classic equivalent | When |
|---|---|---|---|
| Parent → Child (data) | `input()` | `@Input()` | Pass values down |
| Child → Parent (events) | `output()` | `@Output()` + `EventEmitter` | Notify up |
| Two-way | `model()` | `[(x)]` = `[x]` + `(xChange)` | Child reads *and* updates |
| Parent → Child (imperative) | `viewChild()` / template ref | `@ViewChild` | Call child methods, focus, media |
| Projected content → Parent | `contentChild(ren)()` + `ng-content` | `@ContentChild` | Shell/wrapper components |
| Sibling ↔ Sibling | shared **signal** service | shared `BehaviorSubject` service | No binding path between them |
| Event broadcast | shared **Subject** service | same | Transient one-off events |
| Distant / global | shared service / NgRx / router | same | Cross-tree, app-wide state |

---

## 10. Interview soundbites

- **Data down, events up** is Angular's default flow: `input()` down, `output()` up.
- **`[(x)]` is not magic** — it desugars to `[x]="v"` + `(xChange)="v = $event"`; `model()` wires both.
- **`@ViewChild` vs `@ContentChild`:** view = own template; content = projected via `ng-content`.
- **Why a service for siblings?** They share the *same* DI singleton, so one writes and the other reads — no parent relay needed.
- **Signal service vs Subject service:** signal for current-value state; Subject for fire-once events. `BehaviorSubject` bridges the two worlds pre-signals.
- **Avoid over-using `viewChild`** — reach for it only when declarative inputs/outputs can't express the interaction (imperative control, DOM measurement, third-party widgets).
