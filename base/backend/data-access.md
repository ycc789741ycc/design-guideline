# Data Access

## Principles

- The domain and use cases reach persistence only through a repository
  interface defined with the domain model — never through an ORM, ODM,
  query builder, or driver directly. See [Repositories](#repositories).
- Transactions wrap any operation that must be atomic; partial writes on
  failure are not acceptable.
- Migrations are versioned, forward-only in production (no editing a
  migration that has already run in any shared environment), and reviewed
  like any other code change.
- Migrations run to completion before the application starts serving, via
  the `migrate` step that `start-app` depends on — never lazily on first
  request, and never from application startup code racing across
  replicas. See [`../shared/build-and-run.md`](../shared/build-and-run.md).
- The migration runner is a one-off container started from the
  application image, attached to the infra network — not a CLI installed
  on a developer's machine or a CI runner. The runner, driver, and
  schema history are then identical locally, in CI, and in the deploy
  job.
- Because migrations land before the new code does, each one is
  backward-compatible with the currently running version of the
  application.

## Repositories

Persistence depends on the domain, not the other way round: the domain
declares what it needs as a repository interface, and the persistence code
implements it.

```
bookstore/orders/
├── __init__.py                       # public API (+ factory taking infra handles)
├── _order.py                         # domain entity
├── _order_repository.py              # repository interface — domain types only
├── _cancel_order.py                  # use case — depends on the interface
└── _sqlalchemy_order_repository.py   # implementation — ORM ↔ domain mapping
```

- **The interface is defined with the domain model.** Each aggregate root
  (`Order`) has a repository interface (`OrderRepository`) that lives next
  to it in the component's domain code — a `typing.Protocol` (or ABC) in
  Python, an `interface` in TypeScript.
- **It speaks only domain types.** Parameters and return values are
  entities, value objects, and primitives — never ORM models, ODM
  documents, sessions, cursors, query builders, or driver types. Methods
  are named in the domain's language (`find_by_id`, `save`,
  `find_overdue_for(customer_id)`), not a generic `query(filter)` or
  `Repository[T]` that leaks query semantics to callers.
- **Domain and use cases depend on the interface only.** They never import
  the ORM/ODM, the driver, or a concrete repository.
- **The implementation lives in the component's persistence code**, named
  after its technology (`SqlAlchemyOrderRepository`,
  `MongoOrderRepository`). It is the only code that imports the ORM/ODM,
  and it maps records to domain models and back — ORM models and documents
  never leave it. Like the rest of the component, it is private.
- **The composition root wires it.** The component's entry point exposes a
  factory that takes infrastructure handles (a session factory, a client)
  and builds the implementation; `api/main.py` calls that factory. Nothing
  else constructs a repository.
- **Tests follow the tiers.** Domain and use-case tests pass an in-memory
  fake of the interface and run in `make test-unit`; the concrete
  repository is tested against the real datastore in
  `make test-integration`
  (see [`../shared/testing-philosophy.md`](../shared/testing-philosophy.md)).
- **Enforce it in `make lint`**: an import rule forbids ORM/ODM and driver
  imports outside persistence modules (e.g. an import-linter contract, or
  `no-restricted-imports` scoped by file pattern).

Why: see [ADR 0005](../../docs/decisions/0005-reach-persistence-through-domain-defined-repository-interfaces.md).
Full example: [`examples/good-service.md`](examples/good-service.md).

## Query patterns

- Avoid N+1 queries — batch or join instead of looping with individual
  queries.
- Long-running or large-result queries are paginated; never load an
  unbounded result set into memory.
- Indexes are added deliberately, with the query pattern that motivates them
  documented in the migration.

## Schema conventions

- Table and column names follow [`shared/naming-conventions.md`](../shared/naming-conventions.md)
  (snake_case).
- Every table has `created_at` / `updated_at` timestamps unless there's a
  specific reason not to.
- Foreign keys are enforced at the database level, not just in application
  code.

## Caching

- Cache invalidation strategy is documented alongside the cache itself —
  don't add a cache without stating how and when it's invalidated.
- Cached data is allowed to be stale only where the domain explicitly
  tolerates it (documented per use case).
