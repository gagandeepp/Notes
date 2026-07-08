# RxJS in Angular — Beginner to Advanced (with Interview Focus)

> A progressive guide: start from *why reactive programming exists*, build up each core primitive (**Observable, Observer, Subscription, Operators, Subject, Schedulers**) explaining the **need** each one fills, then move through operators, real-life Angular use cases, testing, and interview Q&A.
>
> **Modern context (Angular 16+ / v22):** signals now handle *synchronous UI state*; RxJS remains the tool for *streams, events, and complex async*. Knowing where the boundary is is itself a common interview question — covered in §16.
>
> **Coming from Python:** an Observable is a lazy, cancellable, push-based stream — closer to a generator you subscribe to than a Promise. Operators compose like `itertools`/`toolz`, but over *time*.

---

# PART 1 — FOUNDATIONS (Beginner)

## 1. Why RxJS exists — the problem it solves

Before reactive programming, async code in JS was handled with **callbacks** and **Promises**. Both have limits that RxJS was built to fix.

**Callbacks** → nesting hell, no cancellation, hard error handling:
```typescript
getUser(id, (err, user) => {
  if (err) return handle(err);
  getOrders(user.id, (err, orders) => {   // callback pyramid
    if (err) return handle(err);
    // ...
  });
});
```

**Promises** → better, but still limited:
```typescript
const p = fetch('/api/users');   // ❌ eager: runs immediately, can't stop it
// ❌ single value: resolves once, can't model a stream of events
// ❌ not cancellable: no built-in way to abort
// ❌ no operators: no debounce/retry/merge without extra libraries
```

**What we actually need for UIs:**
- Model things that emit **many values over time** (clicks, keystrokes, websocket messages, intervals).
- **Cancel** work that's no longer needed (abort a stale HTTP request).
- **Compose & transform** async flows declaratively (debounce a search, retry on failure, merge streams).
- **Lazy** execution — don't do work until someone actually wants the result.

RxJS answers all four with the **Observable** and a rich set of **operators**. That's the entire motivation.

| | Promise | Observable |
|---|---|---|
| Values | Single | 0..∞ over time |
| Execution | Eager (starts now) | Lazy (starts on subscribe) |
| Cancellable | No | Yes (`unsubscribe`) |
| Operators | No | 100+ (map, filter, retry, …) |
| Re-run | No (cached) | Yes (re-subscribe re-runs) |

---

## 2. Observable — the need & the mechanics

### The need
We need a **single abstraction** that represents "a stream of values arriving over time," works the same whether there's 1 value or 1000, whether sync or async, and can be transformed and cancelled. That abstraction is the **Observable**.

An Observable is **lazy** and **cancellable**: it's just a blueprint of *how* to produce values. Nothing happens until an Observer subscribes.

### Mechanics
```typescript
import { Observable } from 'rxjs';

// A hand-built Observable to see what's really happening:
const numbers$ = new Observable<number>(subscriber => {
  subscriber.next(1);          // push a value
  subscriber.next(2);
  const id = setTimeout(() => {
    subscriber.next(3);        // push later (async)
    subscriber.complete();     // signal "done" — no more values
  }, 1000);

  // teardown: runs on unsubscribe/complete/error — THIS is how cancellation works
  return () => clearTimeout(id);
});

// Nothing has run yet. The producer function fires only now:
const sub = numbers$.subscribe(v => console.log(v));  // 1, 2, (…1s…) 3
```

Three things an Observable can send an Observer:
- `next(value)` — 0 or more times
- `complete()` — at most once; stream ends, teardown runs
- `error(err)` — at most once; stream ends with failure, teardown runs

> Convention: name Observable variables with a trailing `$` (`user$`, `results$`). Interviewers notice it.

### Cold vs Hot (fundamental, always asked)
- **Cold**: the producer starts *per subscriber*. Each subscriber gets its own independent execution. e.g. `http.get()`, `of()`, `interval()`.
- **Hot**: the producer runs independently of subscribers; late subscribers miss earlier values. e.g. DOM events, `Subject`.

```typescript
const cold$ = this.http.get('/api/users');
cold$.subscribe();   // request #1
cold$.subscribe();   // request #2  ← each subscribe re-runs the producer
```
To make a cold Observable behave hot/shared, see `share()` / `shareReplay()` in §11.

---

## 3. Observer — the need & the mechanics

### The need
The Observable *produces* values, but something has to *consume* them and decide what to do with each value, each error, and completion. That consumer is the **Observer** — a bundle of three callbacks. Separating producer (Observable) from consumer (Observer) is what makes streams reusable: one Observable, many Observers, each reacting differently.

### Mechanics
An Observer is just an object with up to three handlers:
```typescript
const observer = {
  next:     (value) => console.log('value:', value),
  error:    (err)   => console.error('error:', err),
  complete: ()      => console.log('done'),
};

source$.subscribe(observer);

// Shorthand — pass just the next callback:
source$.subscribe(v => console.log(v));

// Partial observer is fine:
source$.subscribe({
  next: v => console.log(v),
  error: e => console.error(e),   // omit complete if you don't need it
});
```

The contract: after `error` or `complete`, the Observer receives **nothing more**. This guarantee is why error handling and cleanup are predictable.

---

## 4. Subscription — the need & the mechanics

### The need
When you subscribe, the Observable may start long-lived work — a timer, an event listener, a websocket. If you never stop it, it keeps running after you no longer care (memory leak, callbacks firing on destroyed components). The **Subscription** is the handle that lets you **cancel** that execution and trigger the Observable's teardown.

### Mechanics
```typescript
import { interval } from 'rxjs';

const sub = interval(1000).subscribe(n => console.log(n));  // 0,1,2,…
// later:
sub.unsubscribe();   // stops emissions AND runs the Observable's teardown fn

// Bundle multiple subscriptions and tear them all down at once:
const bag = new Subscription();
bag.add(stream1$.subscribe());
bag.add(stream2$.subscribe());
bag.unsubscribe();   // cancels both
```

This is the piece Promises lack — **cancellation** — and it's central to avoiding leaks in Angular (§13).

---

## 5. Operators — the need & pipe

### The need
Raw `next` values are rarely what you want. You need to **transform** (map), **filter**, **combine**, **debounce**, **retry**, etc. — declaratively, without manual state. **Operators** are pure functions that take an Observable and return a new Observable, so you can build a pipeline. `pipe()` chains them left-to-right.

### Mechanics
```typescript
import { map, filter } from 'rxjs/operators';
import { of } from 'rxjs';

of(1, 2, 3, 4)
  .pipe(
    filter(n => n % 2 === 0),  // 2, 4
    map(n => n * 10),          // 20, 40
  )
  .subscribe(console.log);      // 20, 40
```

Operators are **pure and lazy** — they don't run until subscription, and each returns a fresh Observable (the source is untouched). This composability is the whole point of RxJS. Categories covered in Part 2: creation, transformation, filtering, combination, higher-order mapping, error handling, utility.

---

## 6. Subject — the need & the variants

### The need
A plain Observable is **unicast**: each Observer triggers its own independent producer run (cold). But sometimes you need **multicast** — one source, many listeners all sharing the *same* execution and values. Examples: an app-wide event bus, a shared state store, notifying many components of one change. You also sometimes need to **push values imperatively** from outside (e.g. "the user clicked save" → emit). A plain Observable can't do that; its values come only from its producer function.

The **Subject** solves both: it is **both an Observable and an Observer**. You can `subscribe` to it (many times — multicast) *and* call `.next()` on it to push values in from anywhere.

### Mechanics
```typescript
import { Subject } from 'rxjs';

const events$ = new Subject<string>();

events$.subscribe(v => console.log('A:', v));   // listener A
events$.subscribe(v => console.log('B:', v));   // listener B  ← same execution

events$.next('hello');   // A: hello   B: hello   ← multicast to both
events$.next('world');   // A: world   B: world
```

### The four Subject variants (common interview follow-up)

| Subject | Behavior | When you need it |
|---|---|---|
| `Subject` | No initial value; subscribers get only **future** emissions | Event bus, imperative triggers, destroy-notifier |
| `BehaviorSubject` | Requires an **initial value**; new subscribers immediately get the **current** value | State store ("current value" semantics) |
| `ReplaySubject` | Replays the last **N** (or time-windowed) values to new subscribers | Caching recent events for late subscribers |
| `AsyncSubject` | Emits only the **last** value, and only on `complete()` | One-shot final result (rare) |

```typescript
// BehaviorSubject — the classic pre-signals state store
import { BehaviorSubject } from 'rxjs';

class UserStore {
  private user$ = new BehaviorSubject<User | null>(null);   // initial value required
  readonly currentUser$ = this.user$.asObservable();        // expose read-only
  setUser(u: User) { this.user$.next(u); }
  get snapshot() { return this.user$.value; }               // synchronous current value
}
// A brand-new subscriber to currentUser$ immediately receives the latest user.

// ReplaySubject — new subscribers get the last 3 events
const recent$ = new ReplaySubject<string>(3);
recent$.next('a'); recent$.next('b'); recent$.next('c'); recent$.next('d');
recent$.subscribe(console.log);   // b, c, d  (last 3 replayed)

// Subject — a destroy notifier (see §13)
private destroy$ = new Subject<void>();
```

> **Why `.asObservable()`?** It hands out a read-only view so consumers can subscribe but can't call `.next()` and hijack the stream — encapsulation. Interviewers like this detail.

> **Modern note:** for *synchronous component/service state*, Angular now favors **signals** over `BehaviorSubject`. But Subjects remain correct for stream-shaped, event-driven, or multicast scenarios, and are everywhere in existing code. Bridge with `toSignal()` / `toObservable()`.

---

## 7. How the pieces fit together (mental model)

```
Observable  ── defines HOW values are produced (lazy blueprint)
    │  .subscribe(Observer)
    ▼
Observer    ── { next, error, complete }: WHAT to do with each value
    │  returns
    ▼
Subscription ── handle to CANCEL the execution (unsubscribe → teardown)

Operators   ── pure functions that build NEW Observables from existing ones (pipe)
Subject     ── Observable + Observer in one: MULTICAST + imperative push
```

One sentence each for an interview:
- **Observable**: lazy, cancellable stream of values over time.
- **Observer**: the consumer's `next/error/complete` callbacks.
- **Subscription**: the cancellation handle.
- **Operator**: pure function that transforms one Observable into another.
- **Subject**: a multicasting Observable you can also push into.

---

# PART 2 — OPERATORS (Intermediate)

## 8. Creation operators

```typescript
import { of, from, fromEvent, interval, timer, EMPTY, throwError,
         combineLatest, forkJoin, merge, concat, zip, race } from 'rxjs';

of(1, 2, 3);                       // emit given values, then complete
from([1, 2, 3]);                   // from array / iterable / promise
from(fetch('/api'));               // promise → observable
fromEvent(inputEl, 'input');       // DOM events (hot)
interval(1000);                    // 0,1,2,… every 1s (cold)
timer(2000);                       // one emit after 2s
timer(0, 1000);                    // emit now, then every 1s
EMPTY;                             // complete immediately, no values
throwError(() => new Error('x'));  // error immediately
```

### Combination creation operators (frequent topic)

| Operator | Behavior | Real use case |
|---|---|---|
| `forkJoin` | Waits for **all** to complete, emits last of each (like `Promise.all`) | Parallel HTTP calls; render once all arrive |
| `combineLatest` | Emits latest of each whenever **any** emits (after all emitted ≥1) | Reactive filters, dependent form controls |
| `merge` | Interleaves emissions from all sources concurrently | Fold multiple event streams into one |
| `concat` | Runs sources **in order**, next starts when previous completes | Sequential requests/animations |
| `zip` | Pairs emissions **by index** | Correlate streams positionally (rare) |
| `race` | First source to emit **wins**; others discarded | Response vs timeout |

```typescript
// forkJoin — need all three before rendering the dashboard
forkJoin({
  user:   this.api.getUser(id),
  orders: this.api.getOrders(id),
  prefs:  this.api.getPrefs(id),
}).subscribe(({ user, orders, prefs }) => { /* all ready together */ });

// combineLatest — recompute whenever ANY filter changes
combineLatest([this.search$, this.category$, this.sort$]).pipe(
  switchMap(([q, cat, sort]) => this.api.search(q, cat, sort)),
);
```

---

## 9. Transformation & filtering operators

```typescript
import { map, scan, tap, startWith } from 'rxjs/operators';

source$.pipe(map(x => x * 2));                 // transform each value
source$.pipe(scan((acc, x) => acc + x, 0));    // running accumulator (emits each step)
source$.pipe(tap(x => console.log(x)));        // side effect, passes value through
source$.pipe(startWith('initial'));            // seed a value before source emits
```

**`scan` vs `reduce`:** `scan` emits on **every** value (running total); `reduce` emits **once on complete** (final aggregate). Use `scan` for live UI counters, `reduce` for a final sum on a finite stream.

```typescript
import { filter, take, takeUntil, takeWhile, first, last, skip,
         debounceTime, throttleTime, distinctUntilChanged, distinct } from 'rxjs/operators';

source$.pipe(filter(x => x > 10));
source$.pipe(take(3));                     // first 3, then complete
source$.pipe(takeUntil(this.destroy$));    // complete when destroy$ emits
source$.pipe(first(x => x.ready));         // first match, then complete

// The search box, canonical form:
searchInput$.pipe(
  debounceTime(300),          // wait for a typing pause
  distinctUntilChanged(),     // ignore if the value didn't actually change
  filter(q => q.length >= 2), // skip trivial queries
);
```

**`debounceTime` vs `throttleTime`:** debounce emits only after N ms of **silence** (best for "stopped typing"); throttle emits the **first** value then ignores for N ms (best for rate-limiting scroll/resize/click-spam).

---

## 10. Higher-order mapping — the #1 interview question

Each source value triggers an **inner** Observable (typically an HTTP call). These operators differ in how they handle a new source value **while a previous inner is still running**.

| Operator | New value while inner active | Canonical use case |
|---|---|---|
| **`switchMap`** | **Cancels** previous inner, switches to new | Typeahead search, route param → fetch (only latest matters) |
| **`mergeMap`** (`flatMap`) | **Runs concurrently**, no cancellation | Independent parallel work (upload N files) |
| **`concatMap`** | **Queues**; runs after previous completes (preserves order) | Sequential writes/saves |
| **`exhaustMap`** | **Ignores** new until current completes | Login/submit — prevent duplicate submits |

```typescript
// switchMap — cancel stale searches (latest query wins)
this.searchControl.valueChanges.pipe(
  debounceTime(300), distinctUntilChanged(),
  switchMap(q => this.api.search(q)),   // aborts the previous in-flight request
);

// concatMap — save edits strictly in order, never overlapping
this.saveQueue$.pipe(concatMap(change => this.api.save(change)));

// exhaustMap — ignore repeat clicks until login resolves
this.loginClicks$.pipe(exhaustMap(() => this.auth.login(this.form.value)));

// mergeMap — upload files concurrently, cap at 3 at a time
from(files).pipe(mergeMap(file => this.api.upload(file), 3));
```

> Interview mnemonic: **switchMap = latest wins** (search/navigation), **concatMap = order matters** (writes), **exhaustMap = ignore until done** (submit buttons), **mergeMap = all in parallel** (independent tasks). Classic trap: `mergeMap` where you needed `switchMap` → race conditions (a stale response overwrites a fresh one); `switchMap` on writes → can cancel an in-flight save and lose data.

---

## 11. Error handling & multicasting/sharing

### Error handling
```typescript
import { catchError, retry, timeout, finalize } from 'rxjs/operators';
import { of, timer } from 'rxjs';

this.api.getData().pipe(
  timeout(5000),                          // error if nothing arrives in 5s
  retry({ count: 3, delay: (_, n) => timer(500 * 2 ** n) }),  // exponential backoff
  catchError(err => {
    return of([]);                        // recover with a fallback value
    // or: return throwError(() => err);  // rethrow to propagate
  }),
  finalize(() => this.loading = false),   // always runs (like finally)
);
```

**Key rule:** an `error` **terminates** the stream — nothing emits after it. To keep a long-lived source alive (e.g. a search box feeding requests), put `catchError` on the **inner** Observable inside `switchMap`, not the outer stream:

```typescript
this.query$.pipe(
  switchMap(q =>
    this.api.search(q).pipe(catchError(() => of([])))   // inner catch keeps the outer alive
  ),
);
```

### Multicasting / sharing
```typescript
import { share, shareReplay } from 'rxjs/operators';

// Share ONE http execution across multiple template `| async` bindings
readonly config$ = this.http.get<Config>('/api/config').pipe(
  shareReplay({ bufferSize: 1, refCount: true }),
);
```
- `shareReplay(1)` caches & replays the last value → avoids duplicate HTTP calls when several places subscribe.
- `refCount: true` unsubscribes from the source when subscriber count hits 0 → prevents a leaked, never-completing source.

---

# PART 3 — ADVANCED

## 12. Schedulers (advanced, senior interviews)

A **Scheduler** controls **when** and **on what execution context** an Observable emits — sync vs async, microtask vs macrotask, animation frame. Most code never touches them, but they matter for performance and testing.

```typescript
import { observeOn, subscribeOn } from 'rxjs/operators';
import { asyncScheduler, asapScheduler, queueScheduler, animationFrameScheduler } from 'rxjs';

// Emit heavy work asynchronously so it doesn't block the current task:
source$.pipe(observeOn(asyncScheduler));

// Batch DOM-driven emissions to animation frames:
positions$.pipe(observeOn(animationFrameScheduler));
```
Schedulers: `queueScheduler` (sync, queued — recursion-safe), `asapScheduler` (microtask), `asyncScheduler` (macrotask / `setTimeout`), `animationFrameScheduler` (rAF). The `TestScheduler` (§14) uses virtual time to test streams synchronously.

## 13. Unsubscription & memory leaks (guaranteed topic)

Not unsubscribing from a **long-lived** stream (`interval`, `fromEvent`, a `Subject`) leaks memory and fires callbacks on destroyed components. Approaches, best → worst:

```typescript
// 1) async pipe — Angular subscribes AND unsubscribes for you (preferred)
//    template: <div>{{ data$ | async }}</div>

// 2) takeUntilDestroyed() (v16+) — cleanest manual approach
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
constructor() {
  this.stream$.pipe(takeUntilDestroyed()).subscribe(/* … */);
}

// 3) takeUntil(destroy$) — classic pattern
private destroy$ = new Subject<void>();
ngOnInit() { this.stream$.pipe(takeUntil(this.destroy$)).subscribe(); }
ngOnDestroy() { this.destroy$.next(); this.destroy$.complete(); }

// 4) manual Subscription (fine for one; error-prone for many)
private sub = new Subscription();
ngOnInit() { this.sub.add(this.stream$.subscribe()); }
ngOnDestroy() { this.sub.unsubscribe(); }
```
HTTP Observables **complete** after one emission, so they self-clean — but `takeUntilDestroyed` is still safer if the component may be destroyed mid-flight.

## 14. Testing Observables — marble testing (senior follow-up)

`TestScheduler` runs streams in **virtual time**, so async logic tests run synchronously and deterministically. Marble strings describe emissions: `-` = 1 frame, `a`/`b` = values, `|` = complete, `#` = error, `()` = same-frame group.

```typescript
import { TestScheduler } from 'rxjs/testing';

let scheduler: TestScheduler;
beforeEach(() => {
  scheduler = new TestScheduler((actual, expected) => expect(actual).toEqual(expected));
});

it('maps values', () => {
  scheduler.run(({ cold, expectObservable }) => {
    const source$ = cold('-a-b-c|', { a: 1, b: 2, c: 3 });
    const result$ = source$.pipe(map(x => x * 10));
    expectObservable(result$).toBe('-a-b-c|', { a: 10, b: 20, c: 30 });
  });
});

it('debounces', () => {
  scheduler.run(({ cold, expectObservable }) => {
    const source$ = cold('a-b-c---|');
    const result$ = source$.pipe(debounceTime(3, scheduler));
    expectObservable(result$).toBe('------c-|');   // only the settled value survives
  });
});
```

---

# PART 4 — APPLIED

## 15. Real-life Angular use cases (talk through these)

**Typeahead / autocomplete**
```typescript
results$ = this.searchControl.valueChanges.pipe(
  debounceTime(300), distinctUntilChanged(),
  filter(q => q.length >= 2),
  switchMap(q => this.api.search(q).pipe(catchError(() => of([])))),
);
```
**Dependent dropdowns** (country → state)
```typescript
states$ = this.countryControl.valueChanges.pipe(
  switchMap(country => this.api.getStates(country)),
);
```
**Parallel dashboard load**
```typescript
vm$ = forkJoin({ user: this.api.user(), stats: this.api.stats() });
```
**Reactive filters** (multiple controls → one query)
```typescript
list$ = combineLatest([this.search$, this.status$, this.page$]).pipe(
  debounceTime(200),
  switchMap(([q, status, page]) => this.api.list(q, status, page)),
);
```
**Prevent double-submit**
```typescript
this.submit$.pipe(exhaustMap(() => this.api.save(this.form.value)));
```
**Polling**
```typescript
data$ = timer(0, 5000).pipe(switchMap(() => this.api.getStatus()));
```
**Route param → data**
```typescript
item$ = this.route.paramMap.pipe(
  map(p => p.get('id')!),
  switchMap(id => this.api.getItem(id)),
);
```
**loading/error/data view-model** (one async pipe, no flag juggling)
```typescript
vm$ = this.api.getData().pipe(
  map(data => ({ data, loading: false, error: null })),
  startWith({ data: null, loading: true, error: null }),
  catchError(err => of({ data: null, loading: false, error: err })),
);
```

## 16. Signals vs RxJS — where each belongs (modern must-know)

- **Signals**: synchronous UI state, derived values, template bindings. Simpler, no unsubscribe, fine-grained change detection.
- **RxJS**: streams and events over time, complex async orchestration (cancellation chains, retries, combining sources), websockets.
- **Bridge** at the boundary:
```typescript
import { toSignal, toObservable } from '@angular/core/rxjs-interop';

count = toSignal(this.count$, { initialValue: 0 });  // Observable → Signal
query$ = toObservable(this.querySignal);             // Signal → Observable
results = toSignal(
  this.query$.pipe(debounceTime(300), switchMap(q => this.api.search(q))),
  { initialValue: [] },
);
```

---

## 17. Rapid-fire interview Q&A

- **Observable vs Promise?** Observable: lazy, cancellable, multi-value, operator-composable. Promise: eager, non-cancellable, single-value.
- **Why nothing happens without `subscribe`?** Observables are lazy — the producer runs only on subscription.
- **Observer vs Observable?** Observable produces; Observer (`next/error/complete`) consumes.
- **What's a Subscription for?** Cancellation — `unsubscribe()` stops the execution and runs teardown.
- **Why a Subject over an Observable?** Multicast (one execution, many listeners) + imperative `.next()` push.
- **BehaviorSubject vs Subject?** BehaviorSubject has an initial/current value delivered to new subscribers; Subject only sends future values.
- **BehaviorSubject vs ReplaySubject?** BehaviorSubject holds one current value; ReplaySubject buffers/replays the last N.
- **switchMap vs mergeMap?** switchMap cancels the prior inner (latest wins); mergeMap runs all concurrently.
- **When does switchMap cause a bug?** On writes/saves — it can cancel an in-flight mutation; use concatMap/exhaustMap.
- **debounceTime vs distinctUntilChanged?** debounce waits for a pause; distinctUntilChanged drops consecutive duplicates. Combined in search.
- **Cold vs hot?** Cold: per-subscriber execution on subscribe. Hot: shared, subscriber-independent. `shareReplay` makes cold hot.
- **forkJoin vs combineLatest?** forkJoin waits for all to complete then emits once; combineLatest emits on each change after all emitted once.
- **Where does catchError go in switchMap?** On the inner Observable, so an error doesn't kill the outer source.
- **scan vs reduce?** scan emits every step; reduce emits once on complete.
- **How to avoid leaks?** `async` pipe, `takeUntilDestroyed`, or `takeUntil(destroy$)`.
- **How do signals change RxJS usage?** Signals for sync state; RxJS for streams/async; bridge via `toSignal`/`toObservable`.
- **What's a Scheduler?** Controls when/where emissions happen (sync/async/rAF); `TestScheduler` enables virtual-time testing.

---

## 18. Gotchas (senior-level polish)

- **Nested `subscribe` is an anti-pattern** — flatten with `switchMap`/`concatMap` instead of subscribing inside a subscribe.
- **Missing inner `catchError`** kills a long-lived source on the first error.
- **`shareReplay` without `refCount`** can keep a source alive forever (leak) — use `{ bufferSize: 1, refCount: true }`.
- **Side effects in `map`** — keep `map` pure; use `tap` for side effects.
- **Subscribing in a loop/template** — subscribe once, share, or use `async`.
- **Assuming order with `mergeMap`** — it doesn't preserve order; use `concatMap` when order matters.
- **Reusing a completed/errored Subject** — it won't emit again; create a new one.
- **`combineLatest` needs every source to emit once** before it produces anything — seed with `startWith` if one might be silent.
