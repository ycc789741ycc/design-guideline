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
├── _order_repository.py              # repository interface + OrderFilter — domain types only
├── _cancel_order.py                  # use case — depends on the interface
└── _sqlalchemy_order_repository.py   # implementation — ORM ↔ domain mapping
```

- **The interface is defined with the domain model.** Each aggregate root
  (`Order`) has a repository interface (`OrderRepository`) that lives next
  to it in the component's domain code — a `typing.Protocol` (or ABC) in
  Python, an `interface` in TypeScript.
- **It speaks only domain types.** Parameters and return values are
  entities, value objects, and primitives — never ORM models, ODM
  documents, sessions, cursors, query builders, or driver types.
- **Every repository has the same six methods** — `create`, `get`,
  `get_list`, `get_count`, `update`, `delete` (camelCase in TypeScript:
  `getList`, `getCount`, `pageSize`):

  ```python
  class BookRepository(Protocol):
      def create(self, book: Book) -> Book: ...
      def get(self, book_id: BookId) -> Book | None: ...
      def get_list(self, filter: BookFilter, page: int = 1,
                   page_size: int | None = None) -> list[Book]: ...
      def get_count(self, filter: BookFilter) -> int: ...
      def update(self, book: Book) -> Book: ...
      def delete(self, book_id: BookId) -> None: ...

  @dataclass(frozen=True)
  class BookFilter:
      author_id: AuthorId | None = None
      status: BookStatus | None = None
      created_after: datetime | None = None
  ```

  - `create` returns the stored entity with its id and timestamps
    assigned. `get` returns `None` when nothing matches. `update` and
    `delete` report a missing entity as a typed not-found error
    (`BookNotFoundError`), per
    [`../shared/error-handling.md`](../shared/error-handling.md).
- **`get_list` and `get_count` take a filter, one per aggregate.** `<Aggregate>Filter`
  (`UserFilter`, `BookFilter`) is a frozen value type defined next to the
  interface. Every field is optional and defaults to `None`, meaning "don't
  filter on this"; set fields combine with AND, and an empty filter matches
  everything. Fields are named in the domain's language (`customer_id`,
  `is_overdue`, `created_after`), not column names or operators. A new
  query is a new filter field, never a new method.
- **`get_list` sorts newest first by default**: `created_at` descending,
  ties broken by id descending so pages are stable (every table has
  `created_at` — see [Schema conventions](#schema-conventions)).
- **`get_list` paginates by default**: `page=1, page_size=None`. `page` is
  1-based; `page_size=None` returns every match, and then `page` must be
  1. A `page` or `page_size` below 1 is a typed validation error. It
  returns just that page's entities, as a plain list.
- **`get_count(filter)` returns the total number of matches**, ignoring
  pagination — it takes no `page`/`page_size`, because a count limited to
  one page is only `len(items)`. Call it only when a total is actually
  needed (page controls, "N results"); a caller that only needs to know
  whether there is a next page asks `get_list` for `page_size + 1` items
  instead. When the list and the total must agree exactly under concurrent
  writes, run both calls in one read transaction.
- **Pass `page_size=None` only when the filter keeps the set small** (one
  order's lines, one customer's addresses). User-facing lists and anything
  over a table that grows without bound pass a `page_size`.
- **No other methods by default.** An extra method is allowed only for an
  operation the six can't express — an atomic increment, a bulk update, an
  OR query or a non-default order a use case genuinely needs — and the PR
  says why.
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

Why: see [ADR 0006](../../docs/decisions/0006-give-every-repository-the-same-crud-shape.md).
Full example: [`examples/good-service.md`](examples/good-service.md).

## Query patterns

- Avoid N+1 queries — batch or join instead of looping with individual
  queries.
- Long-running or large-result queries are paginated. An unpaginated read
  (`get_list` with `page_size=None`) is only for sets the filter keeps
  small; never load a set that grows without bound into memory.
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
