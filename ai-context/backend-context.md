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
  ├── unit/             → make test-unit; mirrors src/ (api/, bookstore/orders/)
  └── integration/      → make test-integration; mirrors src/ the same way
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
  (`bookstore/money/`). Tests split by tier first (`tests/unit/`,
  `tests/integration/`), then mirror `src/` beneath it
  (`tests/unit/bookstore/orders/`); tier is decided by what the test needs.
- **Layering**: delivery mechanism → component public API; inside a
  component, use case → domain → data access. Dependencies point inward
  only. Domain logic has zero framework/DB dependencies and must be
  unit-testable in isolation; data access depends on the domain (it
  implements the domain's repository interfaces) and is private to its
  component.
- **Module boundaries**: a component is the module. Only call it through
  its entry point — never import another component's private modules. No
  circular dependencies between components. Enforce with an import rule in
  `make lint` (import-linter, `no-restricted-imports`).
- **API design**: resources are plural nouns (`/orders`); non-CRUD actions
  are verb sub-paths (`/orders/{id}/cancel`). Version via URL path or
  header, consistently. Consistent response envelope and pagination
  params across all endpoints. See any org override for protocol
  differences (e.g. gRPC instead of REST).
- **Data access**: domain and use cases reach persistence only through a
  repository interface defined with the domain model — never an ORM, ODM,
  query builder, or driver directly. One interface per aggregate root
  (`OrderRepository` next to `Order`; `Protocol`/ABC in Python,
  `interface` in TypeScript), speaking only domain types (entities, value
  objects, primitives — never ORM models, documents, sessions, cursors)
  with domain-language methods (`find_by_id`, `save`, `find_overdue`), not
  a generic `query(filter)`. The implementation is named after its
  technology (`SqlAlchemyOrderRepository`), is the only code importing the
  ORM/ODM, maps records ↔ domain models, and stays private; the component's
  factory builds it from infra handles the composition root passes in.
  Unit tests use an in-memory fake of the interface; the implementation is
  tested in `make test-integration`. An import rule in `make lint` forbids
  ORM/ODM/driver imports outside persistence modules. Wrap atomic
  operations in transactions. Migrations are forward-only once run in a shared environment and run to
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

Inside the `orders` component: domain object with pure business logic +
`OrderRepository` interface next to it → use case that depends on the
interface, orchestrates, and returns typed `Result` → Postgres
implementation that alone knows the ORM and maps rows ↔ `Order` → a
factory in the component's entry point that wires them from infra handles
passed in by `api/main`. In `api/`: a route that imports only that entry
point and maps the result to HTTP status/response, no business logic.
