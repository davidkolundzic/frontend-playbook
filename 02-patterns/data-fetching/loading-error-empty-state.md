# Loading / Error / Empty State

**Status:** `APPROVED`

---

## Purpose

Every data-fetching UI has four possible states. This document defines how to model, sequence, and display each one consistently across the app.

```
Initial   → no fetch started yet
Loading   → request in flight
Error     → request failed
Success   → data received (may still be an empty list)
```

Modelling these as separate signals — not a single `status` enum — keeps the template readable and the store logic straightforward.

---

## When to Use

- Any component that fetches from an API and renders the result
- Any async action where the user needs feedback (submit, delete, reload)

## When Not to Use

- Local-only UI state (toggle, tab selection) — no async, no loading state needed
- Data from a synchronous store already in memory — no loading state needed

---

## Recommended Default

Three signals in the store. One `computed` for empty. Fixed `@if` priority order in the template.

```
_loading  signal<boolean>
_error    signal<string | null>
_data     signal<T[]>
isEmpty   computed  ← derived from the above, never manually set
```

---

## Code Example

### Store — three signals, one computed

```typescript
// features/invoices/state/invoice.store.ts
import { computed, inject, Injectable, signal } from '@angular/core';
import { Subscription } from 'rxjs';
import { InvoiceService } from '../data/invoice.service';
import { Invoice } from '../models/invoice.model';

@Injectable({ providedIn: 'root' })
export class InvoiceStore {
  private readonly service = inject(InvoiceService);
  private _req: Subscription | null = null;

  private readonly _data    = signal<Invoice[]>([]);
  private readonly _loading = signal(false);
  private readonly _error   = signal<string | null>(null);

  readonly data    = this._data.asReadonly();
  readonly loading = this._loading.asReadonly();
  readonly error   = this._error.asReadonly();

  // empty = loaded successfully but list is empty
  readonly isEmpty = computed(
    () => !this._loading() && !this._error() && this._data().length === 0
  );

  load(): void {
    this._req?.unsubscribe();
    this._loading.set(true);
    this._error.set(null);

    this._req = this.service.getAll().subscribe({
      next: invoices => {
        this._data.set(invoices);
        this._loading.set(false);
      },
      error: (err: Error) => {
        this._error.set(err.message ?? 'Could not load invoices.');
        this._loading.set(false);
      },
    });
  }
}
```

### Component — fixed `@if` priority chain

```typescript
// features/invoices/ui/invoice-list.component.ts
import { Component, inject, OnInit } from '@angular/core';
import { InvoiceStore } from '../state/invoice.store';

@Component({
  selector: 'app-invoice-list',
  standalone: true,
  template: `
    @if (store.loading()) {
      <p>Loading...</p>
    } @else if (store.error()) {
      <p class="error">{{ store.error() }}</p>
      <button (click)="store.load()">Retry</button>
    } @else if (store.isEmpty()) {
      <p>No invoices yet.</p>
    } @else {
      @for (invoice of store.data(); track invoice.id) {
        <p>{{ invoice.id }}</p>
      }
    }
  `,
})
export class InvoiceListComponent implements OnInit {
  protected readonly store = inject(InvoiceStore);

  ngOnInit() {
    this.store.load();
  }
}
```

---

## State Machine

| State | `loading` | `error` | `data` | `isEmpty` |
|---|---|---|---|---|
| Initial | `false` | `null` | `[]` | `true` |
| Loading | `true` | `null` | `[]` | `false` |
| Error | `false` | `'msg'` | `[]` | `false` |
| Empty success | `false` | `null` | `[]` | `true` |
| Success | `false` | `null` | `[…]` | `false` |

> **Note:** Initial and Empty success have the same signal values. To distinguish them (e.g. to suppress the empty state before the first load), add a `_loaded = signal(false)` flag and set it to `true` after the first successful response.

---

## `@if` Priority — Why This Order Matters

```
1. loading   ← check first — prevents flashing error or empty state during a fetch
2. error     ← check before empty — a failed request must never look like "no data"
3. isEmpty   ← only after we know there is no error and no load in progress
4. else      ← render the data
```

Swapping the order causes silent bugs:
- `isEmpty` before `error` → a failed request with empty `_data` shows "No items" instead of the error message
- `isEmpty` before `loading` → empty state flashes during the initial fetch before data arrives

---

## Mistakes to Avoid

| Mistake | Why it's wrong |
|---|---|
| Single `status: 'idle' \| 'loading' \| 'error' \| 'success'` enum | Derived state like `isEmpty` is awkward; cannot have error + stale data simultaneously |
| `isEmpty = data.length === 0` without checking `loading` | Shows empty state while the fetch is still in flight |
| `error` check after `isEmpty` | Failed response with empty data shows "No data" instead of the error |
| Not resetting `_error` before a new fetch | Stale error message flashes while new data loads |
| `_loading.set(false)` only in `next` | Spinner stays on screen permanently after a failed request |
| No retry button on the error state | User has no recovery path without a full page reload |

---

## Interview Answer

> "I model async state with three signals — `loading`, `error`, and `data` — plus a `computed` `isEmpty` that is true only when loading is false, error is null, and the list is empty. In the template I use a fixed `@if` priority chain: loading first, then error with a retry button, then empty state, then the data. This ordering prevents state conflicts and makes every async boundary in the UI explicit and recoverable without any extra logic."
