# Naming Conventions

## General rules

- Names should describe *what* something is or does, not *how* it's
  implemented (e.g. `getActiveUsers()` not `filterUsersByLoopingArray()`).
- Avoid abbreviations unless they're domain-standard and in the
  [glossary](glossary.md) (e.g. `id`, `url` are fine; `usrCfg` is not).
- Booleans read as a yes/no question: `isActive`, `hasPermission`,
  `canRetry` — not `active`, `permission`, `retry`.
- Functions are verbs or verb phrases: `calculateTotal()`, `sendInvoice()`.
- Collections are plural: `users`, `orderItems` — not `userList`, `orderArr`.

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
