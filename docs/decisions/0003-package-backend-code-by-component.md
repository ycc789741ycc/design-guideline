# 0003. Package backend code by component

- **Status:** Superseded by [0004](0004-package-backend-code-by-component-with-tests-split-by-tier.md)
- **Date:** 2026-09-27
- **Deciders:** Design guideline maintainers
- **Supersedes:** [0002](0002-group-the-domain-model-in-one-domain-folder-split-by-feature.md)

## Context

[ADR 0002](0002-group-the-domain-model-in-one-domain-folder-split-by-feature.md)
put the domain model in one top-level `domain/` folder split by feature,
next to separate `application/` and `presentation/` layer folders. It
recorded the cost at the time: one business capability is spread across
several top-level folders. In practice that cost dominates. Working on
orders means touching `domain/order/`, the matching use cases, and the
matching repositories in different places. Nothing in the layout lets a
capability hide its entities or repository from the rest of the code, so
any module can reach into any other's internals. And extracting a
capability into its own service means collecting it from every layer
folder.

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
```

- **Delivery mechanism** (`api/`): framework code, routes, request/response
  schemas, auth, and the composition root (`main.py`) that wires components
  to infrastructure. It is named after the delivery type; a second one
  (a queue worker, a CLI) is a sibling (`worker/`, `cli/`), never nested in
  the application.
- **Application** (`<product>/`): named after the product or business,
  not a technical bucket. It contains no framework code.
- **Component** (`<product>/<component>/`): owns its domain model, use
  cases, and data access. Its public API is its package entry point
  (`__init__.py`; `index.ts` in TypeScript); every other module in it is
  private (`_`-prefixed in Python, unexported in TypeScript). Nothing
  outside the component imports anything but that entry point.
- Components don't form cycles, and there is no catch-all
  `common/`/`shared/`/`utils/` package — a concept several components use
  becomes its own component.
- `tests/` mirrors `src/` (`tests/api/`, `tests/<product>/`).

The rule itself lives in
[`base/backend/architecture.md`](../../base/backend/architecture.md#project-layout-package-by-component).

## Consequences

**Easier**

- Work on one capability stays inside one folder.
- A component's internals are private, so a component can change its
  entities, queries, or storage without touching any caller.
- "No framework code in business logic" is checkable against one path
  (`src/<product>/**`), and the public-API boundary against one pattern.
- Adding or replacing a delivery mechanism (a worker next to the HTTP
  API, a different web framework) doesn't touch the application.
- Extracting a component into its own service is close to a folder move:
  its public API becomes the service contract.

**Harder**

- Layering *inside* a component (use case → domain → data access) is no
  longer visible in the folder tree; it is kept by convention and review.
- The public-API boundary is only real when enforced: each repo needs an
  import rule (import-linter contract, `no-restricted-imports`) wired into
  `make lint`.
- Drawing component boundaries is a design judgement — which subdomain a
  concept belongs to — rather than a mechanical "which layer is this".
- Repos that already moved to `domain/` under ADR 0002 must move again.
- The application package has a different name in every repo, so tooling
  and docs refer to it as `<product>/` rather than a fixed path.

## Alternatives considered

- **Keep ADR 0002 (one `domain/` folder, split by feature, plus layer
  folders).** Lost because the scatter and the lack of encapsulation it
  accepted are exactly the problems this decision fixes.
- **Fully self-contained vertical slices, with routes inside each
  component.** Keeps even more of a capability in one place, but puts
  framework code inside the application, so the business logic can no
  longer be checked or reused independently of the delivery mechanism.
  Lost for that reason.
- **Package by layer (`controllers/`, `services/`, `repositories/`).** The
  most familiar layout, but it scatters every capability across the tree
  and makes everything public to everything. Lost as strictly worse on
  both counts.
- **A generic application folder name (`components/`, `core/`, `app/`).**
  Consistent across repos, but it reads as a technical bucket rather than
  as the application. Lost because the top level should say what the
  system does, not how it's built.
