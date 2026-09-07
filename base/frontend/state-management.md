# State Management

## Categories of state

- **Local UI state** (is a dropdown open, form input values before submit) —
  keep in component state, don't lift to global stores.
- **Server/remote state** (data fetched from an API) — manage with a
  dedicated data-fetching layer (cache, refetch, invalidation), not plain
  global state; treat it as a cache of the server, not a second source of
  truth.
- **Global client state** (theme, current user, feature flags) — only things
  that are genuinely needed in many unrelated places belong in a global
  store.

## Rules

- Don't put server data into global state management as a general habit —
  this leads to stale-cache bugs. Use a data-fetching library's cache
  instead, and let global state hold only client-only concerns.
- Derived state is computed, not stored — don't `useState` a value that can
  be calculated from existing state/props.
- Side effects (network calls, subscriptions) are isolated to well-known
  points (data-fetching hooks, dedicated effect hooks) — not scattered
  through render logic.

## Anti-patterns

- Prop-drilling more than 2-3 levels — use composition or a scoped context
  instead.
- A single global store holding everything (UI state, server cache, and
  domain state undifferentiated) — this makes it impossible to reason about
  what invalidates what.
