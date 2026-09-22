# Backend Architecture

## Layers

Read [`shared/principles.md`](../shared/principles.md) first. Backend code is
organized into layers with a strict dependency direction — outer layers
depend on inner layers, never the reverse:

```
presentation (controllers/handlers)
        ↓
application (use cases / orchestration)
        ↓
domain (core business logic, entities)
        ↓
data access (repositories, persistence)
```

- **Domain layer** has no dependency on frameworks, databases, or HTTP —
  it should be testable in complete isolation.
- **Application layer** orchestrates domain logic to fulfill a use case; it
  doesn't contain business rules itself.
- **Data access layer** is the only place that knows about SQL/ORM/DB
  specifics — swapping databases should not require touching domain logic.

## Domain folder layout

The domain model lives in one top-level folder named `domain/`, split into
one subfolder per feature:

```
src/
└── domain/
    ├── order/
    │   ├── order.ts
    │   ├── order-item.ts
    │   └── order-status.ts
    ├── billing/
    │   ├── invoice.ts
    │   └── payment-term.ts
    └── money/
        └── money.ts
```

- **Name it `domain/`** — not `models/`, `entities/`, or `core/`. `models/`
  in particular reads as ORM models, which belong in the data access layer.
- **One subfolder per feature**, named after the feature in kebab-case
  (`order/`, `billing/`), matching the feature/module names used elsewhere
  in the codebase. No domain file sits loose in `domain/` itself.
- **No catch-all folder** (`common/`, `shared/`, `utils/`). A concept that
  several features use — a value object like `Money` — gets its own
  feature folder (`domain/money/`) that the others depend on.
- **Feature folders follow the module-boundary rules below**: import
  another feature's domain only through its public interface, and never
  create a cycle between two features.
- Everything under `domain/` obeys the domain-layer rule above — no
  framework, database, or HTTP imports.

Why a single top-level folder rather than a `domain/` inside each feature
module: see [ADR 0002](../../docs/decisions/0002-group-the-domain-model-in-one-domain-folder-split-by-feature.md).

## Module boundaries

- A module's public interface is explicit (exported functions/types); internal
  details are not reachable from outside the module.
- Cross-module calls go through the public interface only — never reach into
  another module's internals because it's "just easier."
- Circular dependencies between modules are not allowed. If two modules need
  each other, extract the shared logic into a third module both depend on.

## Granularity

- **Atomic**: a single-responsibility function, validator, or utility.
- **Composite**: a service or use case built from atomic pieces.
- **Feature/module**: a self-contained slice (e.g. `checkout`, `billing`).
- **System/service**: an independently deployable unit.

See [`service-patterns.md`](service-patterns.md) for composite/system-level
guidance.
