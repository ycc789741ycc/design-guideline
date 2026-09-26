# 0005. Reach persistence through repository interfaces defined with the domain model

- **Status:** Accepted
- **Date:** 2026-09-27
- **Deciders:** Design guideline maintainers

## Context

The backend guideline required "all DB access through a repository layer"
but said nothing about who owns the repository's contract or what types
cross it. Its layer diagram drew `domain ↓ data access`, so the domain
appeared to depend on persistence. Under that reading, a use case could
import the ORM or ODM directly, a repository could return ORM rows or
documents, and a domain rule could be written against a table's columns.
The domain would then be testable only with a database, and a schema or
engine change would ripple into business logic. [ADR
0004](0004-package-backend-code-by-component-with-tests-split-by-tier.md)
already puts a component's domain, use cases, and data access together
behind one public API. What it doesn't fix is the direction of the
dependency inside the component.

## Decision

The domain and use cases reach persistence only through a repository
interface that is defined with the domain model. The dependency on
persistence is inverted.

- Each aggregate root has one repository interface (`OrderRepository`),
  declared next to the entity in the component's domain code: a
  `Protocol`/ABC in Python, an `interface` in TypeScript.
- The interface speaks only domain types (entities, value objects,
  primitives), with methods named in domain language. No ORM model, ODM
  document, session, cursor, query builder, or driver type appears in it.
- Domain and use-case code imports the interface, never the ORM/ODM, a
  driver, or a concrete repository.
- The implementation (`SqlAlchemyOrderRepository`) lives in the
  component's persistence code. It is the only code that imports the
  ORM/ODM, it maps records to domain models in both directions, and it is
  private to the component. The component's factory builds it from
  infrastructure handles that the composition root passes in.
- Unit tests substitute an in-memory fake for the interface. The
  implementation is tested against the real datastore in the integration
  tier. An import rule in `make lint` forbids ORM/ODM and driver imports
  outside persistence modules.

The rule itself lives in
[`base/backend/data-access.md`](../../base/backend/data-access.md#repositories).

## Consequences

**Easier**

- Domain and use cases are unit-testable with a fake repository, with no
  database and no network, which is exactly what `make test-unit` requires.
- Schema changes, query tuning, and even a change of storage engine stay
  inside one implementation class per aggregate.
- ORM leakage is mechanical to spot: any ORM/ODM import outside a
  persistence module fails lint.
- The interface states what the domain actually needs from storage,
  instead of offering everything the ORM offers.

**Harder**

- There are two models per aggregate, the domain entity and the ORM model
  or document, plus mapping code between them that must be written,
  tested, and kept in step with the schema.
- ORM conveniences stop at the boundary: lazy loading, identity maps, and
  automatic change tracking don't reach the domain, so loading and saving
  have to be explicit.
- Every new query the domain needs is a new interface method and a new
  implementation, rather than an inline query.
- Transaction scope spanning several repositories needs its own
  abstraction (for example a unit of work), which this decision doesn't
  define.
- Existing code that uses ORM models as entities, or calls the ORM from
  use cases, has to be refactored to comply.

## Alternatives considered

- **ORM models as the domain model (active record).** It needs the least
  code and uses the ORM fully. It lost because business rules then depend
  on the database and the ORM, and the domain can't be tested without
  them. That breaks the domain-layer rule the guideline already has.
- **A generic repository (`Repository[T]` with `query(filter)` or
  similar).** It needs one implementation for all aggregates. It lost
  because it leaks query semantics to callers and isn't domain language,
  so the domain ends up coupled to the storage model anyway.
- **An interface defined in the persistence code.** It keeps storage
  concerns in one place. It lost because the domain would then import
  from persistence, which keeps the dependency pointing the wrong way.
- **Use cases calling the ORM directly, with no repository.** It is the
  most direct approach. It lost because it mixes orchestration with query
  code and leaves nothing to fake in unit tests.
