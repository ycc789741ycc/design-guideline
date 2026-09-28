# 0007. Package backend code by component, with unprefixed private modules

- **Status:** Accepted
- **Date:** 2026-09-28
- **Deciders:** Design guideline maintainers
- **Supersedes:** [0004](0004-package-backend-code-by-component-with-tests-split-by-tier.md)

## Context

[ADR 0004](0004-package-backend-code-by-component-with-tests-split-by-tier.md)
makes a component's package entry point (`__init__.py`, `index.ts`) its
public API, and in Python also requires every other module in the component
to be `_`-prefixed (`_order.py`, `_cancel_order.py`). The prefix marks the
same boundary twice: the entry point already says what is public, and the
import rule in `make lint` already stops anything outside the component from
importing past it. The prefix adds noise to every file name and internal
import, differs from the TypeScript layout for no reason, and has to be
added or removed whenever a module's visibility changes, on top of the
entry-point edit that actually changes it.

Everything else in ADR 0004 stands. This record restates it in full with
the prefix removed, so that one record is in force.

## Decision

Backend source has two kinds of top-level package — the delivery
mechanism and the application — and the application is split into
components, one per subdomain or business capability:

```
src/
├── api/              → delivery mechanism
└── bookstore/        → the application (named after the product/business)
    ├── orders/       → component ≈ a subdomain / business capability
    └── customers/    → component ≈ a subdomain / business capability
tests/
├── unit/             → make test-unit (hermetic)
│   ├── api/          → mirrors src/
│   └── bookstore/
│       └── orders/
└── integration/      → make test-integration (infra up and migrated)
    ├── api/          → mirrors src/
    └── bookstore/
        └── orders/
```

- **Delivery mechanism** (`api/`): framework code, routes, request/response
  schemas, auth, and the composition root (`main.py`) that wires components
  to infrastructure. A second delivery mechanism (a queue worker, a CLI) is
  a sibling (`worker/`, `cli/`), never nested in the application.
- **Application** (`<product>/`): named after the product or business, not
  a technical bucket. It contains no framework code.
- **Component** (`<product>/<component>/`): owns its domain model, use
  cases, and data access. Its public API is its package entry point
  (`__init__.py`; `index.ts` in TypeScript): what it imports and lists in
  `__all__` (or re-exports) is public, and every other module in the
  component is private. Private modules keep plain names (`order.py`, not
  `_order.py`) in both languages. Nothing outside the component imports
  anything but that entry point, and an import rule in `make lint`
  enforces it.
- Components don't form cycles, and there is no catch-all
  `common/`/`shared/`/`utils/` package. A concept several components use
  becomes its own component.
- **Tests split by tier first, then mirror `src/`**: `tests/unit/` and
  `tests/integration/` are the top level, and each mirrors `src/` beneath
  it. A test's tier is still decided by what it needs, and its path under
  the tier by the code it covers.

The rule itself lives in
[`base/backend/architecture.md`](../../base/backend/architecture.md#project-layout-package-by-component).

## Consequences

**Easier**

- All of ADR 0004's benefits carry over: work on one capability stays in
  one folder, "no framework code in business logic" is checkable against
  `src/<product>/**`, delivery mechanisms can be added or swapped, a
  component can be extracted by moving its folder, and test tiers are
  selected by directory.
- Module names read as plain domain names (`order.py`,
  `sqlalchemy_order_repository.py`), and Python and TypeScript components
  look the same.
- Making something public or private is one edit, in the entry point; no
  file renames.

**Harder**

- ADR 0004's costs carry over: layering inside a component is kept by
  convention, component boundaries are a design judgement, the tests for
  one component are split across two tier trees, and repos on ADR 0002 or
  0003 must move.
- Nothing in a Python file name says a module is private, which goes
  against the common PEP 8 habit of `_`-prefixing internals. Editors will
  offer `bookstore.orders.order` in auto-import, so the import rule in
  `make lint` becomes the only guard. It must cover every component: a
  `forbidden` contract per component (`bookstore.orders.*` from `api` and
  the other components), which grows with the component count.
- Repos that already `_`-prefixed their private modules under ADR 0004
  rename them.

## Alternatives considered

- **Keep the `_` prefix (ADR 0004).** Visible privacy in the file name
  and in tooling that honours the convention. Lost because it duplicates
  the entry point and the lint rule, and costs a rename on every visibility
  change.
- **Prefix only subpackages (`orders/_persistence/`).** Fewer renames. Lost
  because it leaves two conventions for the same boundary, and a reader
  still can't trust a plain name to be public.
- ADR 0004's own alternatives — no tier folders, mirror first and tier
  second, tier by marker — and ADR 0003's — a single `domain/` folder,
  vertical slices with routes inside components, package by layer, and a
  generic application folder name — lost for the reasons recorded there,
  and those reasons are unchanged.
