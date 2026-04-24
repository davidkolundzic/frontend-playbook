# HttpClient vs httpResource vs rxResource vs resource

**Status:** `APPROVED` for HttpClient · `UNDER EVALUATION` for httpResource / rxResource / resource

---

## Purpose

Angular 21 ships four ways to fetch or load async data. This document explains what each one is, when it makes sense, and when to reach for the default instead.

---

## Quick Comparison

| | `HttpClient` | `httpResource` | `rxResource` | `resource` |

|---|---|---|---|---|

| Returns | `Observable<T>` | `ResourceRef<T>` (signal) | `ResourceRef<T>` (signal) | `ResourceRef<T>` (signal) |
| Loading signal | Manual | Built-in `.isLoading()` | Built-in `.isLoading()` | Built-in `.isLoading()` |
| Error signal | Manual | Built-in `.error()` | Built-in `.error()` | Built-in `.error()` |
| Trigger | Imperative | Reactive (signal → URL) | Reactive (signal → Observable) | Reactive (signal → Promise) |
| Data source | HTTP only | HTTP only | Any Observable | Any Promise / async fn |
| Uses interceptors | Yes | Yes | N/A | **No** |
| Cancel on retrigger | Manual (`unsubscribe`) | Automatic | Automatic | Automatic |
| Stability | Stable | Experimental (Angular 21) | Experimental (Angular 21) | Experimental (Angular 21) |
| Use in production | Yes | With caution | With caution | With caution |

---

## When to Use Each

### `HttpClient` — default choice

**Use when:**

- Any standard data fetching in a production app
- You need full control over loading/error state
- The fetch is triggered imperatively (button, `ngOnInit`, user action)
- You need retry, debounce, or custom error handling

**Do not use when:** You want zero-boilerplate reactive fetching tied directly to signal inputs — that is `httpResource`'s job.

```typescript
@Injectable({ providedIn: 'root' })
export class ProductService {
  private readonly http = inject(HttpClient);

  getAll(): Observable<Product[]> {
    return this.http.get<Product[]>('/api/products');
  }
}
```

---

### `httpResource` — reactive fetch, experimental

**Use when:**

- The URL or params are derived from signals (route params, filters)
- You want automatic re-fetch when those signals change
- Loading/error state should be zero-boilerplate
- You accept experimental API risk

**Do not use when:**

- Production stability is required — API is not yet stable
- You need fine-grained error handling or retry logic
- The fetch is purely imperative

```typescript
import { httpResource } from '@angular/common/http';

@Component({
  standalone: true,
  template: `
    @if (product.isLoading()) { <p>Loading...</p> }
    @else if (product.error()) { <p>Error loading product.</p> }
    @else { <p>{{ product.value()?.name }}</p> }
  `,
})
export class ProductDetailComponent {
  private readonly route = inject(ActivatedRoute);

  private readonly productId = toSignal(
    this.route.paramMap.pipe(map(p => p.get('id')!))
  );

  // re-fetches automatically when productId signal changes
  readonly product = httpResource<Product>(
    () => `/api/products/${this.productId()}`
  );
}
```

---

### `rxResource` — reactive fetch from any Observable, experimental

**Use when:**

- The data source is an existing typed Observable (e.g. from a service method)
- You want signal-based loading/error without managing them manually
- You are comfortable with experimental APIs

**Do not use when:**

- Same constraints as `httpResource` — experimental, no production stability guarantee
- You need complex RxJS pipeline logic — use `HttpClient` + `switchMap` instead

```typescript
import { rxResource } from '@angular/core/rxjs-interop';

@Component({
  standalone: true,
  template: `
    @if (orders.isLoading()) { <p>Loading...</p> }
    @else { <p>{{ orders.value()?.length }} orders</p> }
  `,
})
export class OrderSummaryComponent {
  private readonly service = inject(OrderService);
  private readonly statusFilter = signal<string>('pending');

  readonly orders = rxResource({
    request: this.statusFilter,
    loader: ({ request: status }) => this.service.getByStatus(status),
  });
}
```

---

---

### `resource` — reactive loader for any async source, experimental

`resource` is the most general of the four. It accepts **any async function** (Promise, `fetch()`, third-party SDK) and wraps the result in the same `ResourceRef<T>` signal shape as `httpResource` and `rxResource`.

**Status:** `UNDER EVALUATION` — experimental in Angular 21

**Use when:**

- The data source is a Promise or async function — not an Observable, not `HttpClient`
- You are wrapping a third-party SDK, `fetch()`, or any non-Angular async call reactively
- You want the same reactive signal shape as `httpResource`/`rxResource` without Angular's HTTP layer
- You accept experimental API risk

**Do not use when:**

- The source is an `HttpClient` Observable — use `httpResource` instead (retains interceptors)
- The source is any Observable — use `rxResource` instead
- You need Angular interceptors (auth headers, error normalisation, retry) — `resource` bypasses them
- Production stability is required

```typescript
import { resource, signal, input } from '@angular/core';

@Component({
  standalone: true,
  template: `
    @if (user.isLoading()) { <p>Loading...</p> }
    @else if (user.error()) { <p>Failed to load user.</p> }
    @else { <p>{{ user.value()?.name }}</p> }
  `,
})
export class UserCardComponent {
  readonly userId = input.required<string>();

  // re-runs loader automatically when userId input changes
  readonly user = resource({
    request: this.userId,
    loader: async ({ request: id }) => {
      const res = await fetch(`/api/users/${id}`);
      if (!res.ok) throw new Error('Failed to fetch user');
      return res.json() as Promise<User>;
    },
  });
}
```

> **Warning:** `resource` uses raw `fetch()` — it bypasses Angular's `HttpClient` interceptors. Auth headers added by an interceptor will not be included. If you need interceptors, use `httpResource` instead.

## Recommended Default

**Use `HttpClient` in a Service → Store → Component pattern for all production code.**

`httpResource`, `rxResource`, and `resource` are compelling for reactive parameter-driven fetching, but their APIs are experimental. Track them — when they stabilise they will likely replace a lot of manual loading/error boilerplate.

See [httpclient-service-store-component.md](httpclient-service-store-component.md) for the production default.

---

## Mistakes to Avoid

| Mistake | Why it's wrong |

|---|---|
| Using `httpResource` in a production feature | Experimental — breaking changes likely |
| Putting `httpResource` inside a store | It is already reactive — wrapping it adds no value |
| Using `rxResource` instead of a `switchMap` pipeline | Loses RxJS operator composability |
| Mixing `httpResource` and manual signal state | Redundant — `httpResource` already has `.isLoading()` and `.error()` |
| Not scoping `httpResource` to a component injector | App-level resource keeps fetching even after component is destroyed |
| Using `resource` when you need Angular interceptors | `resource` uses raw `fetch()` — auth headers from interceptors are not applied |

---

## Interview Answer

> "`HttpClient` is the stable default — it returns a typed Observable and gives full control over error handling and state. `httpResource`, `rxResource`, and `resource` are Angular 21 experimental APIs that all return a `ResourceRef<T>` — a signal with built-in `.isLoading()` and `.error()`. The difference is the data source: `httpResource` wraps `HttpClient` calls and respects Angular interceptors, `rxResource` wraps any Observable, and `resource` wraps any Promise or async function but bypasses interceptors. I use `HttpClient` in production and track `httpResource` for when it stabilises."
