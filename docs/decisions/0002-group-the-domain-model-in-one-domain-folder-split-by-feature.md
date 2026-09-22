# 0002. Group the domain model in one `domain/` folder, split by feature

- **Status:** Accepted
- **Date:** 2026-09-22
- **Deciders:** Design guideline maintainers

## Context

The backend guideline defined a domain layer — core business logic with no
framework, database, or HTTP dependency — but not where it lives on disk.
The only hint was `domain/order.ts` in one example. Repos were left to
pick between `domain/`, `models/`, `entities/`, and `core/`, and between
one global folder and a domain folder inside each feature module
(`order/domain/`). Each choice is reasonable alone; mixing them across
repos means a reader has to relearn the layout, and a review can't point
at one rule to say where a new entity goes.

## Decision

The domain model lives in one top-level folder named `domain/`, split into
one kebab-case subfolder per feature (`domain/order/`, `domain/billing/`).
No domain file sits loose in `domain/`, and there is no catch-all
`common/`/`shared/`/`utils/` folder: a concept several features use gets
its own feature folder (`domain/money/`). Feature folders are modules and
follow the existing module-boundary rules — public interface only, no
cycles.

The rule itself lives in
[`base/backend/architecture.md`](../../base/backend/architecture.md#domain-folder-layout).

## Consequences

**Easier**

- One place to look for business rules in every repo; the layer boundary
  is visible in the tree, so a framework or ORM import under `domain/`
  stands out in review.
- The domain can be tested, linted, and dependency-checked as one unit
  (e.g. a rule forbidding `domain/**` from importing a framework).
- Feature subfolders still keep related entities together, so `domain/`
  doesn't become one flat folder of unrelated classes.

**Harder**

- A single feature is spread across layer folders (`domain/order/`,
  `application/…`, `presentation/…`), so working on one feature means
  touching several top-level folders, and extracting a feature into its
  own service means collecting it from each of them.
- Feature names must be kept in step across layers by convention; nothing
  in the layout forces `domain/order/` and the matching application code
  to use the same name.
- Existing repos using `models/`, `entities/`, or per-feature `domain/`
  folders need a move to comply — mechanical, but it touches every import
  of the domain.

## Alternatives considered

- **A `domain/` folder inside each feature module (`order/domain/`).**
  Keeps a feature self-contained and easy to extract, but scatters the
  domain layer across the tree, so "is the domain free of framework
  imports?" can no longer be checked against one path. Lost because the
  layer boundary is the rule we most want visible and enforceable.
- **A flat `domain/` with no feature subfolders.** Simplest, but grows
  into one large folder where ownership and cross-feature dependencies are
  invisible. Lost because it gives up the module boundaries the guideline
  already requires.
- **`models/` or `entities/` as the folder name.** Common in MVC and ORM
  ecosystems, but `models/` there usually means ORM-backed classes, which
  contradicts a domain layer with no database dependency. Lost because the
  name would suggest the wrong thing belongs in it.
