# Backend Context (condensed from base/backend/)

Read `shared-context.md` first — applies here too.

- **Layering**: presentation → application → domain → data access.
  Dependencies point inward only. Domain layer has zero framework/DB
  dependencies and must be unit-testable in isolation.
- **Module boundaries**: only call other modules through their public
  interface. No circular dependencies. No reaching into internals.
- **API design**: resources are plural nouns (`/orders`); non-CRUD actions
  are verb sub-paths (`/orders/{id}/cancel`). Version via URL path or
  header, consistently. Consistent response envelope and pagination
  params across all endpoints. See any org override for protocol
  differences (e.g. gRPC instead of REST).
- **Data access**: all DB access through a repository layer, never raw
  queries in business logic. Wrap atomic operations in transactions.
  Migrations are forward-only once run in a shared environment and run to
  completion before the app serves (the `migrate` step `start-app` depends
  on) — never lazily on first request or from startup code racing across
  replicas; each is backward-compatible with the running version. Avoid
  N+1 queries — batch/join instead.
- **Service patterns**: one use case per class/function, named after the
  action. Prefer async messaging between services over sync calls where
  possible; sync calls need timeouts + circuit breakers. Any retryable
  operation must be idempotent.
- **Configuration**: externalized as env vars declared in `.env`, never
  hardcoded and never baked into the build artifact. Read once at startup
  into a validated typed config object; required values (environment-
  specific settings, secrets) have no default and fail startup when
  missing.

## Example pattern (see base/backend/examples/good-service.md for full code)

Domain object with pure business logic → application-layer use case that
orchestrates + returns typed `Result` → presentation-layer controller that
only maps to HTTP status/response, no business logic.
