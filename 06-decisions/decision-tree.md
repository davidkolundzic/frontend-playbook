# Decision Tree — State & Async (Angular 20–21)

**Status:** `APPROVED`

> Meta-guide. Use this **before** opening any pattern document.
> Tells you which primitive (`signal`, `computed`, `effect`, `toSignal`, `HttpClient`, `httpResource`, `rxResource`, `resource`, raw RxJS) belongs in your hand for the situation at hand — and whether the choice is safe to ship.

This document does not redefine patterns. It points you at the right one based on **two axes**:

1. **What kind of state / async are you modelling?** (the technical shape of the problem)
2. **What is the project mode?** (production-stable vs. innovation sandbox)

For the patterns themselves, follow the cross-links to `02-patterns/` and `06-decisions/approved defaults.md`.

---

## 1. The Two-Axis Model

### Axis 1 — Risk Tier of the Primitive

Every Angular API used in this playbook falls into one of three tiers. The tier dictates what kind of project may use it.

| Tier | Meaning | Examples (Angular 21) | Allowed in Production | Allowed in Sandbox |
|---|---|---|---|---|
| **Stable** | Public, documented, no breaking changes expected | `signal`, `computed`, `effect`, `inject`, `HttpClient`, `toSignal`, `takeUntilDestroyed`, `input()`, `model()`, standalone, functional guards, typed reactive forms | Yes — default | Yes |
| **Developer Preview** | Public, but Angular team explicitly reserves the right to change the API | `linkedSignal` (state of late 2025), signal-based forms preview | Only with an isolation boundary (see §6) | Yes |
| **Experimental** | Shipped, marked experimental, semantics or API may change | `httpResource`, `rxResource`, `resource` | **No** unless wrapped (see §6) | Yes |

> **Rule of thumb:** the stability of the primitive sets the upper bound on where it may live. A production feature may **only** depend on stable APIs at its public boundary. Experimental APIs are allowed inside a single, replaceable file.

### Axis 2 — Project Mode

| Mode | Description | Default bias |
|---|---|---|
| **Production** | Live customers, SLAs, real money, real PII. Deploy cadence measured in weeks. | Reach for the **most boring** option that solves the problem. Stable tier only across module boundaries. |
| **Innovation Sandbox** | Spike, internal tool, demo, learning project, prototype. No SLA. | Reach for the **newest** option that solves the problem. Experimental tier is fine — that is the point. |
| **Production with Pilot** | Production codebase that hosts one isolated experiment to evaluate a new API in real conditions. | Stable everywhere except the pilot module, which is sandboxed and replaceable. |

The same problem can have a different "right answer" in Production and Sandbox. That is not inconsistency — it is risk-pricing.

---

## 2. Decision Tree — State

Use this tree first. It chooses **how state is held** before you decide how it is fetched.

```text
Q1. Is the state shared between more than one component
    OR does it survive a route change?
    ├── No → component-local
    │       ├── Q1a. Does it just toggle / select / hold a single value?
    │       │       → signal()                                   [Stable, default]
    │       └── Q1b. Is it derived from another signal?
    │               → computed()                                 [Stable, default]
    │
    └── Yes → cross-component / feature state
            ├── Q2. Is it async (loaded from an API)?
            │       → go to §3 Async tree, then wrap in a Store.
            │       Default Store shape:
            │         _data signal<T> + _loading + _error  +  computed isEmpty
            │         (see HttpClient + Signals Pattern)
            │
            └── Q3. Is it purely client-side (cart, theme, drafts)?
                    ├── Production           → signal() in @Injectable Store
                    ├── Sandbox / pilot      → linkedSignal() if the value
                    │                          mirrors and overrides another signal
                    │                          [Developer Preview]
                    └── Stream-shaped logic  → keep RxJS (Subject / BehaviorSubject)
                                               only when you need operator pipelines
                                               (debounce, retry, switchMap, merge)
```

**Why no `BehaviorSubject` as a default anymore:** for plain "current value + setter + change notification", `signal()` is shorter, integrates with the template natively, and avoids the `.subscribe()` / `async` pipe split. `BehaviorSubject` survives only where the value is genuinely a *stream* — see §4.

### Quick decisions

| Symptom | Use | Tier |
|---|---|---|
| Single value, mutated by user | `signal<T>()` | Stable |
| Value derived from one or more signals | `computed(() => …)` | Stable |
| Side effect tied to signal change (logging, persistence) | `effect(() => …)` | Stable |
| Mirror an upstream signal but allow local override | `linkedSignal()` (sandbox), or `signal()` + manual sync (production) | Dev Preview / Stable |
| State that is genuinely a stream (multicast, replay, time-windowed) | `BehaviorSubject` / `ReplaySubject` + `toSignal()` at the leaf | Stable |

---

## 3. Decision Tree — Async / Data Fetching

Run this **after** the State tree. It picks the fetch primitive.

```text
Q1. Is the source HTTP, going through Angular's stack (interceptors, etc.)?
    │
    ├── No, it is a raw Promise / third-party SDK / WebSocket
    │   ├── Promise / fetch / SDK call
    │   │   ├── Production → wrap in a Service that returns Observable<T>
    │   │   │                via from(promise) and treat as RxJS source
    │   │   └── Sandbox    → resource({ loader: async () => … })
    │   │                    [Experimental — bypasses interceptors]
    │   │
    │   └── WebSocket / SSE / long-lived stream
    │       → RxJS only. Do NOT convert the live stream to a signal at the source.
    │         At the leaf component you may use toSignal() to read the latest value.
    │
    └── Yes, HTTP through HttpClient
        │
        ├── Q2. Is the URL / params derived from one or more signals
        │       AND should the call re-fire whenever those signals change?
        │   │
        │   ├── No (imperative trigger — ngOnInit, button click, form submit)
        │   │   → HttpClient in Service → Store → Component             [Stable, default]
        │   │     See: 02-patterns/data-fetching/httpclient-service-store-component.md
        │   │
        │   └── Yes (reactive parameter-driven fetch)
        │       │
        │       ├── Production
        │       │   → HttpClient + manual signal-driven re-fetch in the Store.
        │       │     Use effect() OR rxjs switchMap on toObservable(paramSignal).
        │       │     This keeps you on the Stable tier today, easy to migrate
        │       │     to httpResource the day it stabilises.
        │       │
        │       ├── Production with Pilot
        │       │   → httpResource INSIDE one feature's Store, never in a component.
        │       │     The Store re-exports plain signals so the rest of the app
        │       │     does not see the experimental type.                [Experimental]
        │       │
        │       └── Sandbox
        │           → httpResource directly in the component is fine.    [Experimental]
        │             Same story for rxResource when the source is an Observable.
        │
        └── Q3. Need retry, debounce, exponential backoff, request/response shaping,
                or coordination between multiple endpoints?
            → HttpClient + RxJS pipeline. Resource APIs are the wrong tool here —
              they hide the operators you need.
```

### Cross-reference table

| You said… | Production | Sandbox |
|---|---|---|
| "Load list on page load" | `HttpClient` in Service → Store → Component | same, or `httpResource` in component |
| "Re-fetch when route param changes" | `HttpClient` + `toObservable(paramSignal)` + `switchMap` in Store | `httpResource(() => \`/api/x/${id()}\`)` |
| "Search-as-you-type with debounce" | `HttpClient` + `debounceTime` + `switchMap` | same — debounce is RxJS's job either way |
| "Retry with exponential backoff" | `HttpClient` + `retry({ delay })` | same |
| "Wrap a third-party SDK that returns Promise" | Service returns `Observable<T>` via `from()` | `resource({ loader })` |
| "Read a config that is loaded once and never changes" | `APP_INITIALIZER` + `signal()` set on bootstrap | same, or `toSignal(service.getConfig(), { initialValue: null })` |
| "WebSocket pushing live prices" | RxJS stream + `toSignal()` at the leaf | same |

---

## 4. Decision Tree — Stream → Signal Interop

`toSignal` and `toObservable` exist precisely so you do not have to pick one model for the entire app. Use them deliberately.

```text
Source is an Observable. You want to read it from a template / computed.

Q1. Does the UI need to display loading or error state from this source?
    │
    ├── Yes → Do NOT use toSignal here.
    │         Subscribe in a Store action, drive _loading / _error / _data signals
    │         manually. toSignal collapses error into a thrown ErrorEvent which is
    │         clumsy to render.
    │         See: HttpClient + Signals Pattern.
    │
    └── No → toSignal(source$, { initialValue }) is correct.
            Examples that fit:
              • read-only config / feature flags
              • route params / queryParams
              • i18n translations
              • last value of a WebSocket stream

Source is a signal. You want to feed it into an RxJS pipeline.

Q2. Is the pipeline doing operator work that signals can't (debounce, switchMap,
    retry, mergeMap, audit, sample)?
    │
    ├── Yes → toObservable(signal) is correct. Pipe and re-enter the signal world
    │         at the end if needed via toSignal.
    │
    └── No  → don't bridge. Use computed() / effect() instead.
              Bridging for a simple map() is over-engineering.
```

### `toSignal` rules

1. Always pass `initialValue` (or use `requireSync: true` when the source is a `BehaviorSubject` you control).
2. Never call `toSignal` outside an injection context — pass `{ injector }` explicitly if you must.
3. The signal emits `undefined` until the first value if `initialValue` is omitted — this almost always becomes a template bug.
4. Errors are surfaced by **throwing on read**. If the stream can fail, route it through a Store with explicit `_error` instead.

---

## 5. Risk-Tier Rules in Practice

A single rule, phrased three ways:

> **The public boundary of any production feature must depend only on Stable-tier APIs.**
> **Experimental APIs are allowed only behind a Service or Store that returns plain signals or Observables.**
> **If Angular renames the experimental API tomorrow, only one file changes.**

### Allowed (Production with Pilot)

```typescript
// features/orders/state/order.store.ts
import { httpResource } from '@angular/common/http'; // EXPERIMENTAL — contained here
import { computed, inject, Injectable, signal } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class OrderStore {
  private readonly _filter = signal<string>('pending');

  // experimental API — only this file knows about it
  private readonly _resource = httpResource<Order[]>(
    () => `/api/orders?status=${this._filter()}`
  );

  // public surface is plain stable signals
  readonly orders   = computed(() => this._resource.value() ?? []);
  readonly loading  = this._resource.isLoading;
  readonly error    = computed(() => this._resource.error()?.message ?? null);

  setFilter(f: string) { this._filter.set(f); }
}
```

### Not allowed (Production)

```typescript
// features/orders/ui/order-list.component.ts
import { httpResource } from '@angular/common/http';

@Component({ /* ... */ })
export class OrderListComponent {
  // ❌ experimental API leaks into the component layer
  readonly orders = httpResource<Order[]>(() => '/api/orders');
}
```

If the API breaks, every component that imported it has to change. Keep it inside a Store.

---

## 6. Anti-Patterns Specific to "Picking the Wrong Tier"

| Anti-pattern | Why it bites |
|---|---|
| Adopting `httpResource` everywhere because it is shorter | Pre-1.0 API. A single rename forces a fleet-wide refactor. |
| Adopting `signal()` and *also* keeping a parallel `BehaviorSubject` "in case" | Two sources of truth. Pick one — usually the signal. |
| Using `toSignal` on a stream whose UI must display errors | Errors throw on read; templates have no clean way to render them. |
| Using `effect()` to perform HTTP calls | `effect` is for syncing state to the outside world, not for owning async lifecycle. Put fetches in Store actions. |
| `resource({ loader: () => fetch(...) })` for an authenticated endpoint | `resource` skips Angular interceptors — auth headers will not be applied. |
| Wrapping `httpResource` *inside* another store's `signal` | The resource is already reactive. You are duplicating state. |
| Using `linkedSignal` in a production module to "future-proof" | It is in Developer Preview. Future-proofing with a non-stable API is a contradiction. |

---

## 7. Migration Triggers

Stable choices today are not stable choices forever. Revisit when:

1. **An experimental API graduates to Stable.** When `httpResource` / `rxResource` / `resource` lose the experimental marker in Angular's release notes, demote `HttpClient + manual signals` from "default" to "fallback for non-trivial pipelines" and update `approved defaults.md`.
2. **A pattern document moves from `UNDER EVALUATION` to `APPROVED`.** That is the trigger to migrate pilot features to the new default.
3. **A library you depend on switches its public API to signals or resources.** Match the boundary — don't translate at every call site.
4. **You upgrade a major Angular version.** Re-read this document and the version-availability matrix. Some "experimental" rows become "stable" with no other change required.

---

## 8. Version Availability Matrix (Angular 16 → 21)

For mixed-version environments. The team works on 20–21, but acquisitions and consultancy projects sometimes pull older codebases in. This matrix lets you answer "can I use X here?" without reading release notes.

| Primitive | 16 | 17 | 18 | 19 | 20 | 21 |
|---|---|---|---|---|---|---|
| `signal`, `computed`, `effect` | Dev Preview | Stable | Stable | Stable | Stable | Stable |
| `toSignal`, `toObservable` | Stable | Stable | Stable | Stable | Stable | Stable |
| `takeUntilDestroyed`, `DestroyRef` | Stable | Stable | Stable | Stable | Stable | Stable |
| `inject()` (function) | Stable | Stable | Stable | Stable | Stable | Stable |
| Standalone components | Stable | Default | Default | Default | Default | Default |
| New control flow `@if` / `@for` / `@switch` | — | Stable | Stable | Stable | Stable | Stable |
| Functional `CanActivateFn` etc. | Stable | Stable | Stable | Stable | Stable | Stable |
| `input()` / `output()` (signal-based) | — | 17.1 Stable | Stable | Stable | Stable | Stable |
| `model()` (two-way signal binding) | — | 17.2 Stable | Stable | Stable | Stable | Stable |
| `viewChild` / `contentChild` as signals | — | 17.3 Stable | Stable | Stable | Stable | Stable |
| `linkedSignal` | — | — | — | Dev Preview | Dev Preview | Dev Preview |
| `resource()` | — | — | — | Experimental | Experimental | Experimental |
| `rxResource()` | — | — | — | Experimental | Experimental | Experimental |
| `httpResource()` | — | — | — | Experimental | Experimental | Experimental |
| Signal-based forms | — | — | — | — | Preview | Preview |

> Treat this as living. When you upgrade, check Angular's release notes and bump rows that have moved from Preview to Stable.

---

## 9. The 30-Second Cheat Sheet

```text
PRODUCTION DEFAULT
────────────────────────────────────────────────────────
Component-local toggle / value     → signal()
Derived value                       → computed()
Sync side effect                    → effect()
Single fetch on load / button       → HttpClient → Store → Component
Re-fetch on signal change           → HttpClient + toObservable + switchMap (in Store)
Search / debounce / retry           → HttpClient + RxJS operators (in Store)
Read-only static stream             → toSignal(source$, { initialValue })
Live stream (WebSocket)             → RxJS in service, toSignal at the leaf

SANDBOX / PILOT
────────────────────────────────────────────────────────
Anything from Production            → still works
Reactive HTTP from a signal         → httpResource(() => `…${id()}`)
Reactive call from any Observable   → rxResource({ request, loader })
Reactive call from a Promise / SDK  → resource({ request, loader })
Mirror-with-override state          → linkedSignal()

NEVER (any mode)
────────────────────────────────────────────────────────
HttpClient called from a component
Service holding signals
async pipe mixed with signals in the same template
toSignal used as the source of loading/error UI
resource() for an endpoint that needs interceptors
```

---

## 10. Cross-Links

- [Approved Defaults](approved%20defaults.md) — the settled "what" answers
- [HttpClient — Service → Store → Component](../02-patterns/data-fetching/httpclient-service-store-component.md) — production default for fetches
- [HttpClient + Signals Pattern](../02-patterns/data-fetching/HttpClient%20+%20Signals%20Pattern.md) — full deep-dive incl. cancellation & cleanup
- [HttpClient vs httpResource vs rxResource vs resource](../02-patterns/data-fetching/httpclient-vs-httpresource-vs-rxresource.md) — comparison of fetch primitives
- [Loading / Error / Empty State](../02-patterns/data-fetching/loading-error-empty-state.md) — UI state shape
