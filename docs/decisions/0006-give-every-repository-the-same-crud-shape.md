# 0006. Give every repository the same CRUD shape

- **Status:** Accepted
- **Date:** 2026-09-27
- **Deciders:** Design guideline maintainers
- **Supersedes:** [0005](0005-reach-persistence-through-domain-defined-repository-interfaces.md)

## Context

[ADR 0005](0005-reach-persistence-through-domain-defined-repository-interfaces.md)
made the domain reach persistence only through repository interfaces
defined with the domain model. It left the method set open, asking only
for methods named in domain language (`find_by_id`, `save`,
`find_overdue`). In practice, open method sets mean every repository has a
different shape. The interface grows a finder per question. Ordering and
pagination are decided again, differently, in each method. Nothing shared
(fakes, test helpers, list endpoints) can rely on a common contract.

The guideline also said "never load an unbounded result set into memory",
while small, bounded lists (one order's lines, one customer's addresses)
are often wanted whole.

## Decision

Everything in ADR 0005 stays in force:
- The interface is defined with the domain model and speaks only domain
  types.
- Domain and use cases depend on it alone.
- The technology-named implementation is the only code that imports the
  ORM/ODM, maps records to domain models, and stays private behind the
  component's factory.
- Unit tests use fakes, the integration tier tests the implementation, and
  an import rule in `make lint` enforces the boundary.

On top of that, every repository interface has exactly five methods:

```python
class BookRepository(Protocol):
    def create(self, book: Book) -> Book: ...
    def get(self, book_id: BookId) -> Book | None: ...
    def get_list(self, filter: BookFilter, page: int = 1,
                 page_size: int | None = None) -> Page[Book]: ...
    def update(self, book: Book) -> Book: ...
    def delete(self, book_id: BookId) -> None: ...
```

- `create` returns the stored entity. `get` returns `None` when nothing
  matches. `update` and `delete` report a missing entity as a typed
  not-found error.
- `get_list` takes a per-aggregate filter (`UserFilter`, `BookFilter`).
  It is a frozen value type next to the interface. Its fields are optional
  and in domain language, `None` means "don't filter", and set fields
  combine with AND. A new query is a new filter field.
- `get_list` sorts by `created_at` descending, with ties broken by id
  descending.
- `get_list` paginates by default with `page=1, page_size=None`. `page` is
  1-based, and `page_size=None` returns every match. It returns `Page[T]`
  (`items`, `total`, `page`, `page_size`), which lives in its own
  `pagination` component.
- `page_size=None` is only for sets the filter keeps small. User-facing
  lists and tables that grow without bound pass a `page_size`.
- An extra method is allowed only for an operation the five can't express
  (an atomic increment, a bulk update, an OR query, a non-default order),
  and the PR says why.

The rule itself lives in
[`base/backend/data-access.md`](../../base/backend/data-access.md#repositories).

## Consequences

**Easier**

- There is one repository shape to learn, review, and fake. An in-memory
  fake or a shared contract test can be written once and reused per
  aggregate.
- Every list is ordered and paginated the same way, and the `Page` result
  maps directly to a list endpoint's `page`/`limit` parameters and
  response envelope.
- The filter type lists every query the domain makes against an aggregate,
  in one place.
- Small bounded lists can be read whole without a special method.

**Harder**

- Filter types grow as queries accumulate, and each field adds a branch
  to every implementation and fake.
- Queries that don't fit "optional fields, combined with AND" need a
  justified extra method. Examples are OR conditions, ordering by
  relevance, and aggregates. This is judgement the old finder style didn't
  need.
- `total` costs a count query on every `get_list` call.
- Offset pagination drifts on tables with heavy writes (rows shift between
  pages), and it slows down on deep pages.
- `page_size=None` can still load too much if a caller misjudges how small
  the set is. Only review catches that.
- Repositories written under ADR 0005 with domain-named finders have to be
  migrated to the five methods and a filter.

## Alternatives considered

- **Keep ADR 0005's domain-named finders.** They read naturally and grow
  exactly as needed. Lost because every repository ends up with a
  different shape, and pagination and ordering are reinvented per method.
- **A generic `Repository[T]` with an untyped `query(filter)`.** One
  implementation could serve every aggregate. Lost because the filter
  isn't typed per aggregate, so query semantics leak to callers and the
  compiler can't check them.
- **Cursor (keyset) pagination.** It is stable under writes and fast on
  deep pages. Lost because page numbers are what callers and list
  endpoints use, and they are simpler to reason about. A table where
  offset pagination hurts can justify an extra method.
- **Mandatory `page_size`, never unbounded.** It makes an accidental full
  load impossible. Lost because small bounded lists are common, and
  forcing callers to pick an arbitrary size for them adds noise without
  adding safety.
- **Return a plain list, with a separate `count(filter)`.** It skips the
  count when it isn't needed. Lost because most paged lists need the
  total, and one return type keeps callers and fakes uniform.
