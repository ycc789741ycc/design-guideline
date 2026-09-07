# Error Handling

## Principles

- **Fail loudly in development, gracefully in production.** Don't silently
  swallow errors anywhere — either handle them meaningfully or let them
  propagate to a layer that can.
- **Errors should carry context.** An error crossing a boundary (function →
  function, service → service) should include enough information to debug
  without needing to reproduce it locally.
- **Distinguish expected failures from bugs.** A validation failure is not
  the same class of problem as a null pointer — they should be modeled and
  handled differently.

## Patterns

- Use typed/structured errors (custom error classes, `Result`/`Either`
  types, or your language's idiomatic equivalent) rather than throwing raw
  strings or generic `Error`.
- At service/API boundaries, map internal errors to a stable, documented
  set of external error codes — never leak internal stack traces or
  implementation details to clients.
- Retries belong at the layer that knows whether an operation is safe to
  retry (idempotency matters) — don't retry blindly at every layer.

## Anti-patterns

- `catch (e) { }` — swallowing an exception with no logging, no re-throw,
  no handling.
- Catching an error only to immediately re-throw a less specific one,
  losing the original context/stack trace.
- Using exceptions for expected control flow (e.g. throwing to signal "not
  found" when a nullable return or explicit result type would be clearer).

See [`base/backend/examples/anti-patterns.md`](../backend/examples/anti-patterns.md)
for concrete code examples.
