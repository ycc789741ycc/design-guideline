# Naming Conventions

## General rules

- Names should describe *what* something is or does, not *how* it's
  implemented (e.g. `getActiveUsers()` not `filterUsersByLoopingArray()`).
- Avoid abbreviations unless they're domain-standard and in the
  [glossary](glossary.md) (e.g. `id`, `url` are fine; `usrCfg` is not).
- Booleans read as a yes/no question: `isActive`, `hasPermission`,
  `canRetry` — not `active`, `permission`, `retry`.
- Functions are verbs or verb phrases, and the verb says whether the call
  changes state: `getTotal()` only reads, `sendInvoice()` clearly acts. See
  [Read vs. state-changing names](#read-vs-state-changing-names).
- Collections are plural: `users`, `orderItems` — not `userList`, `orderArr`.

## Read vs. state-changing names

A reader must be able to tell from a function's name alone whether calling
it changes anything — without opening the body.

- **Read-only names** (`get…`, `parse…`, `is…`/`has…`/`can…`) promise no
  side effects: they don't write, emit, send, or call anything that does. A
  `getCart()` that creates a cart when none exists is a bug, not a naming
  slip — split it or rename it.
- **State-changing names** use a verb that says so (`create…`, `update…`,
  `delete…`, `execute…`, or outside the domain another plain action verb
  such as `send…`, `publish…`). Changing an object in memory counts as a
  state change.
- Don't hide a write behind a neutral verb (`calculateTotal()` that also
  saves, `handle()`, `process()`, `check()` that mutates).

The backend domain layer holds this to a closed prefix list — see
[`../backend/architecture.md`](../backend/architecture.md#naming-in-the-domain-layer).

## Casing (adjust to match your language's ecosystem convention)

| Element | Convention | Example |
|---|---|---|
| Files | kebab-case | `user-profile.ts` |
| Classes / Types | PascalCase | `UserProfile` |
| Functions / variables | camelCase | `getUserProfile` |
| Constants | SCREAMING_SNAKE_CASE | `MAX_RETRY_COUNT` |
| Database tables/columns | snake_case | `user_profile`, `created_at` |

## Domain terms

Use terms from [`glossary.md`](glossary.md) consistently across backend,
frontend, and docs — a "customer" in the database should be called
"customer," not "client" in one service and "account" in another.
