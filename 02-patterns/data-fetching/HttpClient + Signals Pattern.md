# HttpClient + Signals — Loading / Error / Data Pattern

**Status:** `APPROVED`

> Standard pattern for fetching remote data in Angular 21.
> Service owns transport. Store owns state. Component owns display.

---

## When to Use

| Situation | Approach |

|---|---|
| Fetch on route load | This pattern |
| User-triggered fetch (search, filter, refresh) | This pattern — see Variant A |
| Read-only stream with no error UI needed | `toSignal()` — see Variant B |
| Real-time stream / WebSocket | RxJS only — do not convert to signal |
| Derived value, no async | `computed()` — no HTTP needed |

---

## Architecture

```text
HttpClient  (Service)     ← typed Observable<T>, no state
     ↓
Signal state (Store)      ← _loading / _error / _data signals
     ↓
Component template        ← @if chain: loading → error → empty → data
```

---

## Step 1 — Service

The service is pure transport. It returns `Observable<T>` and never subscribes, holds state, or catches errors.

```typescript
// features/products/data/product.service.ts
import { inject, Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';
import { Product } from '../models/product.model';

@Injectable({ providedIn: 'root' })
export class ProductService {
  private readonly http = inject(HttpClient);
  private readonly base = '/api/products';

  getAll(): Observable<Product[]> {
    return this.http.get<Product[]>(this.base);
  }

  getById(id: string): Observable<Product> {
    return this.http.get<Product>(`${this.base}/${id}`);
  }
}
```

---

## Step 2 — Store

The store owns all state. Three private signals, three public readonly facades, one `computed` for empty state.

The action always:

1. **Aborts the previous in-flight request** before starting a new one
2. Resets `error` and sets `loading` before subscribing
3. Sets `loading = false` in **both** `next` and `error` callbacks

Abort works because `HttpClient` in Angular 21 is `fetch`-based — unsubscribing sends an `AbortSignal` to the browser and cancels the network request.

```typescript
// features/products/state/product.store.ts
import { computed, inject, Injectable, signal } from '@angular/core';
import { Subscription } from 'rxjs';
import { ProductService } from '../data/product.service';
import { Product } from '../models/product.model';

@Injectable({ providedIn: 'root' })
export class ProductStore {
  private readonly service = inject(ProductService);

  // private writable state
  private readonly _data    = signal<Product[]>([]);
  private readonly _loading = signal(false);
  private readonly _error   = signal<string | null>(null);

  // tracks the active request — unsubscribing cancels the HTTP call
  private _request: Subscription | null = null;

  // public readonly facades
  readonly data    = this._data.asReadonly();
  readonly loading = this._loading.asReadonly();
  readonly error   = this._error.asReadonly();

  // derived state — never manually set
  readonly isEmpty = computed(() => !this._loading() && this._data().length === 0);

  loadAll(): void {
    this._request?.unsubscribe(); // cancel previous in-flight request

    this._loading.set(true);
    this._error.set(null);

    this._request = this.service.getAll().subscribe({
      next: products => {
        this._data.set(products);
        this._loading.set(false);
      },
      error: (err: Error) => {
        this._error.set(err.message ?? 'Failed to load products.');
        this._loading.set(false); // ← must be in error path too
      },
    });
  }
}
```

> **Note:** This store is `providedIn: 'root'` (lives for the app lifetime), so `takeUntilDestroyed()` is not needed here. For stores scoped to a component or a route, see the cleanup section below.

---

## Step 3 — Component

The component reads signals and calls store actions. It never touches `Observable` or `HttpClient` directly.

```typescript
// features/products/ui/product-list.component.ts
import { Component, inject, OnInit } from '@angular/core';
import { ProductStore } from '../state/product.store';
import { ProductCardComponent } from './product-card.component';

@Component({
  selector: 'app-product-list',
  standalone: true,
  imports: [ProductCardComponent],
  template: `
    @if (store.loading()) {
      <p>Loading…</p>
    } @else if (store.error()) {
      <p class="error">{{ store.error() }}</p>
      <button (click)="store.loadAll()">Retry</button>
    } @else if (store.isEmpty()) {
      <p>No products found.</p>
    } @else {
      @for (product of store.data(); track product.id) {
        <app-product-card [product]="product" />
      }
    }
  `,
})
export class ProductListComponent implements OnInit {
  protected readonly store = inject(ProductStore);

  ngOnInit() {
    this.store.loadAll();
  }
}
```

The `@if` chain priority is fixed:

| Priority | Condition | Shows |

|---|---|---|
| 1 | `store.loading()` | Spinner / skeleton |
| 2 | `store.error()` | Error message + Retry button |
| 3 | `store.isEmpty()` | Empty state |
| 4 | (else) | Data list |

---

## State Machine

| State | `loading` | `error` | `data` |

|---|---|---|---|
| Initial | `false` | `null` | `[]` |
| Fetching | `true` | `null` | `[]` |
| Success | `false` | `null` | `[...items]` |
| Error | `false` | `'message'` | `[]` |
| Empty | `false` | `null` | `[]` ← `isEmpty() = true` |

---

## Variant A — User-Triggered Fetch

When the fetch fires on user input rather than `ngOnInit`, cancellation is **critical** — the user can type faster than the API responds.

Use a separate `_searchRequest` tracker so search and `loadAll` can be aborted independently.

```typescript
// In the store — add search action with its own subscription tracker
private _searchRequest: Subscription | null = null;

search(query: string): void {
  this._searchRequest?.unsubscribe(); // cancel previous search call

  this._loading.set(true);
  this._error.set(null);

  this._searchRequest = this.service.search(query).subscribe({
    next: results => {
      this._data.set(results);
      this._loading.set(false);
    },
    error: (err: Error) => {
      this._error.set(err.message);
      this._loading.set(false);
    },
  });
}
```

```html
<!-- In the component template -->
<input #q (input)="store.search(q.value)" placeholder="Search products…" />
```

> **Tip:** For debouncing, add `debounceTime` in the store or in the service layer — do not debounce in the template.

---

## Variant A2 — Scoped Store (Component or Route Lifetime)

When a store is provided at component or route level (not `root`), use `takeUntilDestroyed()` to automatically clean up subscriptions when the injector is destroyed.

```typescript
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { DestroyRef, inject } from '@angular/core';

@Injectable() // no providedIn — scoped to component or route
export class ProductStore {
  private readonly service = inject(ProductService);
  private readonly destroyRef = inject(DestroyRef);

  private readonly _data    = signal<Product[]>([]);
  private readonly _loading = signal(false);
  private readonly _error   = signal<string | null>(null);
  private _request: Subscription | null = null;

  readonly data    = this._data.asReadonly();
  readonly loading = this._loading.asReadonly();
  readonly error   = this._error.asReadonly();

  loadAll(): void {
    this._request?.unsubscribe();
    this._loading.set(true);
    this._error.set(null);

    this._request = this.service.getAll()
      .pipe(takeUntilDestroyed(this.destroyRef)) // auto-unsubscribes on destroy
      .subscribe({
        next: products => {
          this._data.set(products);
          this._loading.set(false);
        },
        error: (err: Error) => {
          this._error.set(err.message ?? 'Failed to load.');
          this._loading.set(false);
        },
      });
  }
}
```

> `takeUntilDestroyed()` handles destroy cleanup. Manual `_request?.unsubscribe()` still handles **re-trigger cancellation** — both are needed.

---

## Variant B — `toSignal()` for Read-Only Streams

Use `toSignal()` when you have a stream that should stay live and you do not need to show loading or error state in the UI (config, reference data, lookups).

```typescript
import { toSignal } from '@angular/core/rxjs-interop';

@Injectable({ providedIn: 'root' })
export class ConfigStore {
  private readonly service = inject(ConfigService);

  // Subscribes automatically, cleaned up when injector is destroyed
  readonly config = toSignal(this.service.getConfig(), { initialValue: null });
}
```

**Use when:** The data loads once and never changes, or you only need the latest value.  
**Avoid when:** You need to show a loading spinner, display an error message, or trigger a retry.

---

## Anti-Patterns

| Anti-pattern | Why it's wrong |

|---|---|
| `async` pipe in template | Mixes Observable and signal worlds — pick one |
| `_loading.set(false)` only in `next` | Error path leaves the spinner on screen forever |
| Calling `HttpClient` from the component | Breaks the service → store → component contract |
| `BehaviorSubject` for loading/error state | Replaced by `signal()` in Angular 21 |
| No `_error.set(null)` before a new fetch | Stale error shows while new data is loading |
| Using `any` as the HTTP response type | Defeats TypeScript — always type `http.get<T>()` |
| No `_request?.unsubscribe()` on retrigger | Previous call completes and overwrites newer data |
| Relying only on `takeUntilDestroyed` for cancellation | Handles destroy, not re-trigger — both are needed |
