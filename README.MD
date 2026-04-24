
# Angular Architecture Playbook

A personal decision record and pattern reference for Angular 21 applications.
Opinionated. Practical. Built from real projects.

---

## Stack Defaults

| Concern | Decision |
|---|---|
| Angular version | 21 (standalone, zoneless) |
| State | Signals (`signal`, `computed`, `effect`) |
| DI | `inject()` — no constructor injection |
| Guards | Functional (`canActivateFn`) |
| Forms | Typed Reactive Forms (`FormBuilder`) |
| HTTP | `HttpClient` wrapped in typed services |
| Architecture | Service → Store → Component |
| Routing | Lazy-loaded feature routes |
| Unit tests | Jest |
| E2E tests | Playwright |

---

## Structure

```
01-principles/     Engineering principles and non-negotiables
02-patterns/       Reusable implementation patterns (auth, data-fetching, forms)
03-snippets/       Copy-paste ready code
04-state/          Signal store patterns
05-checklists/     Feature and demo launch checklists
06-decisions/      Approved defaults and ADRs
07-templates/      Blank templates for new patterns and comparisons
```

---

## How to Use This

1. **Starting a new feature?** → [New Feature Checklist](05-checklists/new%20feature%20checklist.md)
2. **Unsure which pattern to use?** → [Approved Defaults](06-decisions/approved%20defaults.md)
3. **Looking for a specific pattern?** → [Index](INDEX.md)
4. **Need copy-paste code?** → `03-snippets/`

---

## Decision Status Labels

| Label | Meaning |
|---|---|
| `APPROVED` | Default choice — use this unless there's a reason not to |
| `SITUATIONAL` | Valid in specific contexts, documented inline |
| `UNDER EVALUATION` | Being assessed — do not use in production |
| `LEGACY` | Replaced — migrate away when possible |
