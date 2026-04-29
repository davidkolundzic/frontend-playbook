# New Feature Checklist

**Status:** `APPROVED`

## Purpose

This checklist helps me start a new feature without hesitation or random architecture choices.

---

## 1. Define the feature type

- Is it read-only?
- Is it CRUD?
- Is it auth-related?
- Is it form-heavy?
- Is it dashboard-like?
- Will it likely grow later?

---

## 2. Check the default

- What is my approved default for this type of feature?
- Is there any strong reason not to use it?
- Am I choosing a different pattern for a good reason, or just because it feels interesting?

---

## 3. Define the data flow

- Where does the data come from?
- Who calls the API?
- Who owns the state?
- Who maps the response?
- What does the component actually need to know?

---

## 4. Define the UI states

- loading
- error
- empty
- data

If data is async, these states should be explicit.

---

## 5. Decide the structure

- Do I need an API service?
- Do I need a store/facade?
- Do I need models?
- Do I need route protection?
- Do I need form validation?

---

## 6. Avoid common mistakes

- Do not put `HttpClient` directly in the component as a default
- Do not mix UI logic and business logic
- Do not skip state structure just to move faster
- Do not pick a new pattern without a clear reason

---

## 7. Reuse what already exists

- Is there already a note for this pattern?
- Is there already a snippet I can reuse?
- Is there already a working example I can copy from?

---

## 8. Practical start order

1. define models
2. define folder structure
3. create API service
4. create store/facade if needed
5. create component
6. handle loading/error/empty/data
7. polish UI

---

## Final question

If I had to explain this feature to a senior developer in English, would the structure still make sense?

If not, simplify it.
