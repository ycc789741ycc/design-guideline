# 0004. Package backend code by component, with tests split by tier

- **Status:** Accepted
- **Date:** 2026-09-27
- **Deciders:** Design guideline maintainers
- **Supersedes:** [0003](0003-package-backend-code-by-component.md)

## Context

[ADR 0003](0003-package-backend-code-by-component.md) packaged backend code
by component: a delivery mechanism (`api/`) and an application package named
after the product, split into components with one public API each. Its test
layout, `tests/` mirroring `src/` directly (`tests/api/`, `tests/<product>/`),
conflicts with an existing rule. Tests run in two tiers, and the standard
targets select a tier by directory: `make test-unit` runs `tests/unit`, and
`make test-integration` runs `tests/integration`
([`base/shared/build-and-run.md`](../../base/shared/build-and-run.md),
[`base/shared/testing-philosophy.md`](../../base/shared/testing-philosophy.md)).
A test under `tests/api/` belongs to neither target, so it would either be
skipped silently or need a recipe that selects tests some other way.

Everything else in ADR 0003 stands. This record restates it in full with the
test layout corrected, so that one record is in force.

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
  (`__init__.py`; `index.ts` in TypeScript). Every other module in it is
  private (`_`-prefixed in Python, unexported in TypeScript), and nothing
  outside the component imports anything but that entry point.
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

- All of ADR 0003's benefits carry over. Work on one capability stays in
  one folder. Component internals are private. "No framework code in
  business logic" is checkable against `src/<product>/**`. A delivery
  mechanism can be added or swapped without touching the application. A
  component can be extracted by moving its folder.
- The test targets keep selecting tiers by directory, with no markers or
  naming conventions, and the tier of any test is visible from its path.
- Finding the tests for a piece of code is still a path lookup, done once
  per tier.

**Harder**

- ADR 0003's costs carry over. Layering inside a component is kept by
  convention. The public-API boundary needs an import rule in
  `make lint`. Component boundaries are a design judgement. Repos on ADR
  0002 must move. The application package's name differs in every repo.
- The tests for one component are split across two trees
  (`tests/unit/<product>/orders/`, `tests/integration/<product>/orders/`),
  and the two mirrors must be kept in step by convention.
- Repos that already laid tests out as `tests/api/` and `tests/<product>/`
  under ADR 0003 must move them under a tier.

## Alternatives considered

- **Keep ADR 0003's layout (`tests/` mirrors `src/`, no tier folders).**
  Lost because the test targets select tiers by directory. Tests outside
  `tests/unit` and `tests/integration` wouldn't run.
- **Mirror first, tier second (`tests/bookstore/orders/unit/`).** Keeps
  all of a component's tests together. Lost because each target would have
  to collect scattered `unit/` folders by glob, which makes it easy to miss
  one and harder to see which tier a test belongs to.
- **Tier by marker or file-name suffix in one mirrored tree.** Needs no
  extra folders. Lost because a missing or wrong marker puts an
  infra-dependent test into the hermetic tier, and nothing in the path
  shows it.
- ADR 0003's own alternatives — a single `domain/` folder, vertical slices
  with routes inside components, package by layer, and a generic
  application folder name — lost for the reasons recorded there, and those
  reasons are unchanged.
