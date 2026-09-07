# API Design

_Default assumes REST over HTTP. An overrides using gRPC, GraphQL, or
another protocol should override this file — see
[`overrides/README.md`](../../overrides/README.md)._

## Endpoint naming

- Resources are nouns, plural: `/orders`, `/users/{id}/invoices`.
- Actions that don't map to CRUD are verbs on a sub-path:
  `/orders/{id}/cancel`, not a `PATCH` with a magic `status` field.
- Consistent pluralization and casing across every service — no
  `/getUser` next to `/orders`.

## Versioning

- Version in the URL path (`/v1/orders`) or a header — pick one and apply it
  everywhere; do not mix strategies across services.
- Breaking changes require a new version; additive changes (new optional
  fields) do not.
- Deprecated versions have a published sunset date before removal.

## Request/response shape

- Consistent envelope across all endpoints (e.g. `{ data, error, meta }`) —
  don't let each endpoint invent its own shape.
- Errors follow the shared error-handling contract
  (see [`shared/error-handling.md`](../shared/error-handling.md)): stable
  error codes, human-readable message, no internal details leaked.
- Pagination, filtering, and sorting use the same query parameter names
  across every list endpoint (`page`, `limit`, `sort`, `filter[...]`).

## Backward compatibility

- Never remove or repurpose a field without a deprecation period.
- Adding required fields to a request is a breaking change — treat it as
  such.
