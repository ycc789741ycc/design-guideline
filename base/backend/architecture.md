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
