# Approved Defaults

> These are the settled decisions. Reach for these first.
> Only deviate with a documented reason.

---

## 1. Component Architecture — Standalone Only

**Status:** `APPROVED`

**Decision:** Every component, directive, and pipe is standalone. No `NgModule`.

**Use when:** Always — this is the Angular 21 default.

**Avoid when:** N/A. Legacy `NgModule`-based code should be migrated.

```typescript
@Component({
  selector: 'app-user-card',
  standalone: true,
  imports: [CurrencyPipe],
  template: `<p>{{ user().name }}</p>`,
})
export class UserCardComponent {
  user = input.required<User>();
}
```

---

## 2. Dependency Injection — `inject()` Only

**Status:** `APPROVED`

**Decision:** Use `inject()` for all dependency injection. No constructor injection.

**Use when:** Every service, token, or function-level DI.

**Avoid when:** Never use constructor DI for new code.

```typescript
@Component({ standalone: true, template: '' })
export class DashboardComponent {
  private readonly userService = inject(UserService);
  private readonly router = inject(Router);
}
```

---

## 3. State — Signals (`signal`, `computed`, `effect`)

**Status:** `APPROVED`

**Decision:** All mutable state is a `signal`. Derived state is `computed`. Side effects use `effect`.

**Use when:** All component and store state. Replaces `BehaviorSubject` in most cases.

**Avoid when:** Streams with complex operators (debounce, switchMap, retry) — use RxJS there.

```typescript
export class CartStore {
  private readonly _items = signal<CartItem[]>([]);

  items = this._items.asReadonly();
  total = computed(() => this._items().reduce((sum, i) => sum + i.price, 0));

  add(item: CartItem) {
    this._items.update(current => [...current, item]);
  }
}
```

---

## 4. Auth Guards — Functional (`canActivateFn`)

**Status:** `APPROVED`

**Decision:** Guards are plain functions, not classes. Injected via `inject()` inside the function body.

**Use when:** All route protection.

**Avoid when:** Class-based `CanActivate` — legacy pattern, do not use.

```typescript
export const authGuard: CanActivateFn = () => {
  const authService = inject(AuthService);
  const router = inject(Router);

  if (authService.isAuthenticated()) return true;
  return router.createUrlTree(['/login']);
};
```

---

## 5. Forms — Typed Reactive Forms

**Status:** `APPROVED`

**Decision:** All forms use `FormBuilder` with explicit types. No `UntypedFormControl`.

**Use when:** Any form with more than one field, or that submits to an API.

**Avoid when:** Single-field ephemeral inputs — a `signal` is simpler there.

```typescript
@Component({ standalone: true, imports: [ReactiveFormsModule] })
export class LoginFormComponent {
  private readonly fb = inject(FormBuilder);

  form = this.fb.group({
    email: ['', [Validators.required, Validators.email]],
    password: ['', Validators.required],
  });

  submit() {
    if (this.form.invalid) return;
    const { email, password } = this.form.getRawValue();
    // email and password are typed as string
  }
}
```

---

## 6. HTTP — Typed Service Layer

**Status:** `APPROVED`

**Decision:** All HTTP calls live in `@Injectable` services. Components never call `HttpClient` directly. Return types are explicit.

**Use when:** Every API interaction.

**Avoid when:** Never put HTTP calls in components or stores directly.

```typescript
@Injectable({ providedIn: 'root' })
export class UserService {
  private readonly http = inject(HttpClient);
  private readonly baseUrl = '/api/users';

  getById(id: string): Observable<User> {
    return this.http.get<User>(`${this.baseUrl}/${id}`);
  }

  update(id: string, payload: Partial<User>): Observable<User> {
    return this.http.patch<User>(`${this.baseUrl}/${id}`, payload);
  }
}
```

---

## 7. Architecture — Service → Store → Component

**Status:** `APPROVED`

**Decision:** Layer responsibilities strictly. Services own HTTP and external IO. Stores own state and derived data. Components own template logic only.

**Use when:** Any feature with shared state or more than one component.

**Avoid when:** Purely local UI state (a toggle, a selected tab) — keep it in the component signal.

```text
UserService          — HttpClient calls, returns Observable<T>
    ↓
UserStore            — signal state, computed values, calls service
    ↓
UserProfileComponent — reads store signals, calls store methods
```

```typescript
// Store
@Injectable({ providedIn: 'root' })
export class UserStore {
  private readonly service = inject(UserService);

  private readonly _user = signal<User | null>(null);
  user = this._user.asReadonly();
  displayName = computed(() => this._user()?.name ?? 'Guest');

  load(id: string) {
    this.service.getById(id).subscribe(u => this._user.set(u));
  }
}

// Component
@Component({ standalone: true, template: `<h1>{{ store.displayName() }}</h1>` })
export class UserProfileComponent {
  protected readonly store = inject(UserStore);

  ngOnInit() {
    this.store.load(inject(ActivatedRoute).snapshot.params['id']);
  }
}
```

---

## 8. Routing — Lazy-Loaded Feature Routes

**Status:** `APPROVED`

**Decision:** Every feature is a lazily loaded route group. No eager-loaded feature components in `app.routes.ts`.

**Use when:** All features. Even small ones — consistency matters.

**Avoid when:** Shared layout components loaded at the root level (e.g. `ShellComponent`).

```typescript
// app.routes.ts
export const routes: Routes = [
  {
    path: 'dashboard',
    loadChildren: () =>
      import('./features/dashboard/dashboard.routes').then(m => m.DASHBOARD_ROUTES),
  },
  {
    path: 'settings',
    canActivate: [authGuard],
    loadChildren: () =>
      import('./features/settings/settings.routes').then(m => m.SETTINGS_ROUTES),
  },
];

// features/dashboard/dashboard.routes.ts
export const DASHBOARD_ROUTES: Routes = [
  { path: '', component: DashboardComponent },
];
```

---

## 9. Unit Tests — Jest

**Status:** `APPROVED`

**Decision:** Jest for all unit and integration tests. No Karma/Jasmine.

**Use when:** Services, stores, pipes, utilities, component logic.

**Avoid when:** Full user-flow testing — use Playwright for that.

```typescript
describe('UserStore', () => {
  let store: UserStore;
  let serviceMock: jest.Mocked<UserService>;

  beforeEach(() => {
    serviceMock = { getById: jest.fn() } as any;
    TestBed.configureTestingModule({
      providers: [UserStore, { provide: UserService, useValue: serviceMock }],
    });
    store = TestBed.inject(UserStore);
  });

  it('sets user on load', () => {
    const user: User = { id: '1', name: 'Alice' };
    serviceMock.getById.mockReturnValue(of(user));
    store.load('1');
    expect(store.user()).toEqual(user);
  });
});
```

---

## 10. E2E Tests — Playwright

**Status:** `APPROVED`

**Decision:** Playwright for all end-to-end and critical user journey tests.

**Use when:** Login flows, checkout, any multi-step user journey.

**Avoid when:** Unit-level logic — that belongs in Jest.

```typescript
// e2e/login.spec.ts
import { test, expect } from '@playwright/test';

test('user can log in and reach dashboard', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill('user@example.com');
  await page.getByLabel('Password').fill('password123');
  await page.getByRole('button', { name: 'Sign in' }).click();
  await expect(page).toHaveURL('/dashboard');
  await expect(page.getByRole('heading', { name: 'Dashboard' })).toBeVisible();
});
```

---

## 11. Signal Forms

**Status:** `GOOD TO KNOW, NOT DEFAULT`

**Decision:** Track signal-based forms, but keep typed Reactive Forms (§5) as the default for any form that ships.

**Use when:** Spikes and prototypes. Internal tools where a future API rename is acceptable.

**Avoid when:** Production features, validation-heavy flows, anything that will be reused across teams. The API is still in preview and will likely change before stabilising.

> When the preview marker is removed in a future Angular release, revisit this section and §5.

---

## 12. RxJS — Use Intentionally

**Status:** `APPROVED` (situational)

**Decision:** RxJS stays in the toolbox. It is not the default for state, but it is the default for streams.

**Use when:**

- Async stream composition (combine two endpoints, coordinate sequence)
- Debouncing and throttling (search-as-you-type)
- Cancellation pipelines (`switchMap`, `takeUntil`)
- Retry with backoff (`retry({ delay })`)
- WebSocket / SSE / any genuinely live stream
- Any place where signal + `effect()` would reinvent an operator that already exists

**Avoid when:**

- "Current value + setter" — that is a `signal()`
- Loading / error / data triple — model as three signals (see [Loading / Error / Empty State](../02-patterns/data-fetching/loading-error-empty-state.md))
- Plain derived value — that is `computed()`

> RxJS is powerful when the problem is a stream. It is overkill when the problem is just state.

---

## 13. Resource APIs (`httpResource` / `rxResource` / `resource`)

**Status:** `GOOD TO KNOW, NOT DEFAULT`

**Decision:** Keep `HttpClient` + Service → Store → Component (§6, §7) as the production default. The resource APIs are tracked, used in sandboxes and pilots, but not the default for a shipping feature.

**Use when:**

- Smaller read-only screens in a sandbox or internal tool
- Signal-first experiments where the URL is derived from a signal
- A pilot module inside a production app, with the resource hidden behind a Store that re-exports plain signals (the public surface stays stable even if the experimental API changes)

**Avoid when:**

- Larger business features
- Anywhere portability across projects matters — the API is experimental in Angular 21
- Endpoints that need interceptors and you are reaching for `resource()` (it bypasses them)

See the full decision tree in [decision-tree.md §3 and §5](decision-tree.md) and the comparison in [HttpClient vs httpResource vs rxResource vs resource](../02-patterns/data-fetching/httpclient-vs-httpresource-vs-rxresource.md).

---

## Architectural Decision Rule

If I am not sure, I start with the approved default and only deviate with a clear reason.

**Clear reasons to deviate:**

- Team standard is different
- Feature is unusually small or simple
- A different pattern genuinely reduces complexity
- A project constraint matters more than personal preference

If none of those apply, the default wins.

---

## Summary

The default mindset:

- Keep architecture simple
- Keep responsibilities clear
- Avoid random variation
- Prefer reusable patterns over improvisation
