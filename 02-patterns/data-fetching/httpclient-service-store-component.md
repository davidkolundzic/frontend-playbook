# HttpClient — Service → Store → Component

**Status:** `APPROVED` — default architecture for all data fetching

---

## Purpose

Defines the standard three-layer data flow for Angular 21 applications.
Each layer has one responsibility and one only.

```text
HttpClient (Service)   ← transport only, typed Observable<T>
       ↓
Feature Store          ← signal state, loading/error/data, actions
       ↓
Component              ← reads signals, calls actions, renders UI
```

---

## When to Use

- Any feature that loads data from an API
- Shared data accessed by more than one component
- Any fetch that needs loading, error, or empty state shown in the UI
- Data that can be reloaded (retry, refresh, filter)

## When Not to Use

- Single-use ephemeral UI state (open/close toggle, selected tab) — use a component `signal()` directly
- Config loaded once at bootstrap — use `APP_INITIALIZER` + `toSignal()`
- Real-time streams — keep as RxJS, do not wrap in this pattern

---

## Recommended Default

Always follow this order: **Service → Store → Component**.
Components never import `HttpClient`. Stores never import `HttpClient`.
Services never hold state.

---

## Code Example

### 1. Model

```typescript
// features/orders/models/order.model.ts
export interface Order {
  id: string;
  total: number;
  status: 'pending' | 'fulfilled' | 'cancelled';
}
```

### 2. Service — transport only

```typescript
// features/orders/data/order.service.ts
import { inject, Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';
import { Order } from '../models/order.model';

@Injectable({ providedIn: 'root' })
export class OrderService {
  private readonly http = inject(HttpClient);
  private readonly base = '/api/orders';

  getAll(): Observable<Order[]> {
    return this.http.get<Order[]>(this.base);
  }

  getById(id: string): Observable<Order> {
    return this.http.get<Order>(`/`);
  }
}
```

### 3. Store — state + actions

```typescript
// features/orders/state/order.store.ts
import { computed, inject, Injectable, signal } from '@angular/core';
import { Subscription } from 'rxjs';
import { OrderService } from '../data/order.service';
import { Order } from '../models/order.model';

@Injectable({ providedIn: 'root' })
export class OrderStore {
  private readonly service = inject(OrderService);
  private _req: Subscription | null = null;

  private readonly _data    = signal<Order[]>([]);
  private readonly _loading = signal(false);
  private readonly _error   = signal<string | null>(null);

  readonly data    = this._data.asReadonly();
  readonly loading = this._loading.asReadonly();
  readonly error   = this._error.asReadonly();
  readonly isEmpty = computed(() => !this._loading() && !this._error() && this._data().length === 0);

  load(): void {
    this._req?.unsubscribe(); // cancel previous in-flight request
    this._loading.set(true);
    this._error.set(null);

    this._req = this.service.getAll().subscribe({
      next: orders => {
        this._data.set(orders);
        this._loading.set(false);
      },
      error: (err: Error) => {
        this._error.set(err.message ?? 'Failed to load orders.');
        this._loading.set(false);
      },
    });
  }
}
```

### 4. Component — display only

```typescript
// features/orders/ui/order-list.component.ts
import { Component, inject, OnInit } from '@angular/core';
import { OrderStore } from '../state/order.store';

@Component({
  selector: 'app-order-list',
  standalone: true,
  template: `
    @if (store.loading()) {
      <p>Loading…</p>
    } @else if (store.error()) {
      <p class="error">{{ store.error() }}</p>
      <button (click)="store.load()">Retry</button>
    } @else if (store.isEmpty()) {
      <p>No orders yet.</p>
    } @else {
      @for (order of store.data(); track order.id) {
        <p>{{ order.id }} — {{ order.status }}</p>
      }
    }
  `,
})
export class OrderListComponent implements OnInit {
  protected readonly store = inject(OrderStore);

  ngOnInit() {
    this.store.load();
  }
}
```

---

## Mistakes to Avoid

| Mistake | Why it's wrong |

|---|---|
| Component injects `HttpClient` directly | Breaks layer separation; untestable |
| Service holds `signal()` state | Service becomes a god object; signals belong in stores |
| Store returns `Observable` to the component | Forces `async` pipe or manual subscribe in component |
| No `_req?.unsubscribe()` before re-fetch | Race condition — slow response overwrites fast one |
| `_loading.set(false)` only in `next` | Error path leaves spinner on screen |
| Untyped `http.get<any>()` | Loses all type safety end-to-end |

---

## Interview Answer

> "In Angular 21 I use a strict three-layer pattern: a typed `HttpClient` service that returns `Observable<T>` with no state, a feature store that manages signal state — loading, error, data — and calls the service, and a standalone component that reads those signals and calls store actions. Components never touch `HttpClient`. Services never hold state. This makes each layer independently testable and keeps the component template purely declarative."
