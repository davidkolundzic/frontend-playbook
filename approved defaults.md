# Approved Defaults

## 1. Data fetching

**Default:** HttpClient -> API Service -> Store/Facade -> Component
**Status:** APPROVED

### I use this for

- business screens
- SaaS features
- dashboards
- list/detail pages
- features that will likely grow

### Why

- clear separation of responsibilities
- easier to maintain
- easier to explain in interviews and team discussions
- stable across different projects and teams

---

## 2. Auth

**Default:** auth-api.service + auth-store + guard + interceptor  
**Status:** APPROVED

### Why I use this

- clear separation of responsibilities
- easier to maintain
- easier to explain in interviews and team discussions
- stable across different projects and teams

### Why

- auth logic stays centralized
- easier to reuse in header, routing, and feature access
- cleaner than scattering auth checks around components

---

## 3. Screen state

**Default:** loading / error / empty / data
**Status:** APPROVED

### I use this for

- any feature that fetches data
- pages with async state
- dashboards and CRUD screens

### Why

- clear UI behavior
- less hidden state logic
- easier to reason about

---

## 4.  Forms

**Default:** Reactive Forms  
**Status:** APPROVED

### I use this for

- login
- create/edit forms
- validation-heavy features
- submit flows

### Why

- predictable
- widely used
- easy to discuss in interviews and on teams

---

## 5. Signal Form

**Status:** GOOD TO KNOW, NOT DEFAULT

## 6. Local UI state

**Default:** Signals  
**Status:** APPROVED

### I use this for

- selected tab
- modal state
- simple filters
- UI toggles
- local view state

### Why

- simple and readable
- good fit for local component state
- avoids unnecessary complexity

---

## 7. RxJS

**Default:** use intentionally, not everywhere  
**Status:** APPROVED

### I use this for

- async stream composition
- debouncing
- cancellation
- combined streams
- advanced async workflows

### Why

- powerful when needed
- not something I want to force into every feature

---

## 8. httpResource

**Default status:** GOOD TO KNOW, NOT DEFAULT

### I use this for

- smaller read-only screens
- signal-first experiments
- simpler fetch scenarios

### Why not default

- not always the clearest choice for bigger business features
- I want my main standard to remain portable and predictable

---


## 9. Architectural Decision Rule

If I am not sure, I start with the approved default and only deviate with a clear reason.

### Clear reasons to deviate

- team standard is different
- feature is unusually small or simple
- a different pattern genuinely reduces complexity
- there is a project constraint that matters more than personal preference

---

## Summary

My default mindset is:

- keep architecture simple
- keep responsibilities clear
- avoid random variation
- prefer reusable patterns over improvisation
