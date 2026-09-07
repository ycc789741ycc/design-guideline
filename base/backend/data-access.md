# Data Access

## Principles

- All database access goes through a repository/data-access layer — no raw
  queries scattered through business logic.
- Transactions wrap any operation that must be atomic; partial writes on
  failure are not acceptable.
- Migrations are versioned, forward-only in production (no editing a
  migration that has already run in any shared environment), and reviewed
  like any other code change.

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
