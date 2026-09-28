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
- An id is named after its entity's full name plus `id`. See
  [Identifier names](#identifier-names).

## Identifier names

An id carries the complete name of the entity it identifies, unshortened,
followed by `id` — so a reader can go from an id to its entity (and back)
without guessing.

| Entity | Correct | Anti-pattern |
|---|---|---|
| `profile` | `profile_id` | `pid`, `prof_id` |
| `skill_assessment` | `skill_assessment_id` | `assessment_id`, `skill_id` |
| `order_item` | `order_item_id` | `item_id`, `line_id` |

- The same holds in every casing: column `skill_assessment_id`, variable
  or field `skillAssessmentId`, type `SkillAssessmentId`, route parameter
  `:skillAssessmentId`.
- Don't drop a word of the entity name because the surrounding context
  seems to make it obvious — `assessment_id` inside a skills module still
  reads as a different entity everywhere it is logged, joined, or passed on.
- When one record references the same entity twice, keep the entity name
  and put the role in front: `author_profile_id` and `reviewer_profile_id`,
  not `author_id` and `reviewer_id`.
- An entity's own primary key column is plain `id`; every reference to it
  from elsewhere uses the full `<entity>_id` form.

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
