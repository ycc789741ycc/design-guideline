# Backend Context (condensed from base/backend/)

Read `shared-context.md` first — applies here too.

- **Project layout (package by component)**:

  ```
  src/
  ├── api/              → delivery mechanism (routes, schemas, auth,
  │                       main.py = composition root)
  └── bookstore/        → the application (named after the product/business)
      ├── orders/       → component ≈ a subdomain / business capability
      │   ├── __init__.py   ← public API (index.ts in TypeScript)
      │   └── _...          ← private modules
      └── customers/    → component ≈ a subdomain / business capability
  tests/
  ├── api/
  └── bookstore/        → mirrors src/
  ```

  The delivery mechanism (`api/`; a worker or CLI is a sibling like
  `worker/`) holds all framework code and wires components to
  infrastructure in the composition root. The application package is named
  after the product — never `app/`, `core/`, `components/`, or `domain/` —
  and contains no framework code. Each component owns its domain model,
  use cases, and data access; everything but its entry point is private
  (`_`-prefixed in Python, not re-exported from `index.ts`). No loose
  modules in the application package and no `common/`/`shared/`/`utils/`
  catch-all — a concept several components use becomes its own component
  (`bookstore/money/`).
- **Layering**: delivery mechanism → component public API; inside a
  component, use case → domain → data access. Dependencies point inward
  only. Domain logic has zero framework/DB dependencies and must be
  unit-testable in isolation; data access is private to its component.
- **Module boundaries**: a component is the module. Only call it through
  its entry point — never import another component's private modules. No
  circular dependencies between components. Enforce with an import rule in
  `make lint` (import-linter, `no-restricted-imports`).
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
  replicas; each is backward-compatible with the running version. The
  migration runner is a one-off container from the app image on the infra
  network, never a host-installed CLI. Avoid
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

Inside the `orders` component: domain object with pure business logic →
use case that orchestrates + returns typed `Result` → exported from the
component's entry point. In `api/`: a route that imports only that entry
point and maps the result to HTTP status/response, no business logic.
