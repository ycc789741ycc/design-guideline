# Backend Architecture

## Project layout: package by component

Read [`shared/principles.md`](../shared/principles.md) first. Backend code is
packaged by component: a delivery mechanism on the outside, and the
application — split into business components — on the inside. Python
example:

```
backend/
├── pyproject.toml
├── src/
│   ├── api/                      # delivery mechanism (FastAPI)
│   │   ├── __init__.py
│   │   ├── main.py               # composition root and app entry point
│   │   ├── routes/
│   │   │   └── orders.py
│   │   ├── schemas.py
│   │   └── auth.py
│   │
│   └── bookstore/                # business components, no framework code
│       ├── __init__.py
│       ├── orders/
│       │   ├── __init__.py       # public API
│       │   └── _...
│       └── customers/
└── tests/
    ├── unit/                     # make test-unit
    │   ├── api/
    │   └── bookstore/
    │       └── orders/
    └── integration/              # make test-integration
        ├── api/
        └── bookstore/
            └── orders/
```

```
src/
├── api/              → delivery mechanism
└── bookstore/        → the application (named after the product/business)
    ├── orders/       → component ≈ a subdomain / business capability
    └── customers/    → component ≈ a subdomain / business capability
```

- **The delivery mechanism** (`api/`) holds everything the framework
  needs: routes, request/response schemas, auth, and the composition root
  (`main.py`) — the one place components are wired to infrastructure
  (database sessions, clients, config). Routes translate between HTTP and a
  component's public API and contain no business rules. A second delivery
  mechanism (queue worker, CLI) is a sibling package (`worker/`, `cli/`),
  never nested inside the application.
- **The application** is one package named after the product or business
  (`bookstore/`) — not `app/`, `core/`, `components/`, or `domain/`. It
  contains no framework code: no web framework, no HTTP types, nothing
  that exists only because of the delivery mechanism.
- **A component** is a subfolder of the application, one per subdomain or
  business capability (`orders/`, `customers/`), named in kebab-case (or
  snake_case where the language requires it for package names). It owns
  its domain model, use cases, and data access.
- **A component's public API is its package entry point** —
  `__init__.py` in Python, `index.ts` in TypeScript. Every other module in
  the component is private: `_`-prefixed in Python, not re-exported from
  `index.ts` in TypeScript. Nothing outside the component — another
  component or the delivery mechanism — imports anything but the entry
  point.
- **No catch-all package** (`common/`, `shared/`, `utils/`). A concept
  several components use — a value object like `Money` — becomes its own
  component (`bookstore/money/`) that the others depend on.
- **No loose modules** in the application package besides its
  `__init__.py`; business code belongs to a component.
- A large component may split into private subpackages (`orders/_domain/`,
  `orders/_persistence/`); they stay behind the same entry point.
- **Tests split by tier first, then mirror `src/`**: `tests/unit/` and
  `tests/integration/` stay the top level (the directories `make test-unit`
  and `make test-integration` run — see
  [`shared/build-and-run.md`](../shared/build-and-run.md)), and
  each mirrors `src/` beneath it: `tests/<tier>/api/` for the delivery
  mechanism, `tests/<tier>/<product>/<component>/` for components. A test's
  tier is decided by what it needs; its path under the tier by the code it
  covers. Component tests go through the public API wherever practical, so
  internals stay free to change.

Why components rather than one `domain/` folder with layer folders around
it: see [ADR 0004](../../docs/decisions/0004-package-backend-code-by-component-with-tests-split-by-tier.md).

## Layers

Dependencies point inward only — from the delivery mechanism into the
application, and within a component toward the domain. The domain sits at
the center and depends on nothing; persistence depends on the domain by
implementing the repository interfaces it declares:

```
delivery mechanism (api/: routes, schemas, auth,       ← outside the application
                    main.py composition root)
        │  public API only
        ▼
use cases (orchestration)                          ┐
        │  uses entities + repository interfaces   │
        ▼                                          │ inside each
domain (entities, rules, repository interfaces)    │ component
        ▲                                          │
        │  implements repository interfaces        │
data access (ORM/ODM repositories, mapping)        ┘
```

- **Domain logic** has no dependency on frameworks, databases, or HTTP —
  it should be testable in complete isolation.
- **Use cases** orchestrate domain logic to fulfill one action; they don't
  contain business rules themselves. They reach persistence only through
  repository interfaces.
- **Data access** is the only code that knows about SQL/ORM/ODM/DB
  specifics. It implements the domain's repository interfaces, maps
  records to domain models, and is private to its component — swapping
  databases should not require touching domain logic or any other
  component. Rules: [`data-access.md`](data-access.md#repositories).
- The composition root (`api/main.py`) supplies infrastructure handles to
  each component's factory, which builds the concrete repositories; domain
  and use cases never construct them.
- These layers are kept inside a component by convention, review, and the
  import rules in `make lint`, not by separate top-level folders.

## Module boundaries

- A component is the module. Its public interface is its package entry
  point; internal details are not reachable from outside it.
- Cross-component calls and delivery-to-component calls go through the
  public interface only — never reach into another component's internals
  because it's "just easier."
- Circular dependencies between components are not allowed. If two
  components need each other, extract the shared concept into a third
  component both depend on.
- Enforce the boundaries mechanically in `make lint`, not only in review:
  e.g. an import-linter contract in Python (`bookstore` may not import the
  web framework; nothing outside a component imports its `_`-prefixed
  modules), or `no-restricted-imports` in TypeScript (no import of
  `bookstore/*/*` except `index`).

## Granularity

- **Atomic**: a single-responsibility function, validator, or utility.
- **Composite**: a service or use case built from atomic pieces.
- **Component**: a subdomain / business capability behind one public API
  (e.g. `orders`, `billing`).
- **System/service**: an independently deployable unit.

See [`service-patterns.md`](service-patterns.md) for composite/system-level
guidance.
