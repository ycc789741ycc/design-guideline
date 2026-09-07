# Service Patterns

Guidance for composite and system-level components — services, use cases,
and the interactions between them.

## Use cases / application services

- Each use case does one thing and is named after it
  (`CancelOrder`, not `OrderService.handleAction(type)`).
- Orchestration logic (calling multiple domain objects/repositories in
  sequence) lives here, not in controllers and not in domain entities.

## Inter-service communication

- Prefer asynchronous messaging for anything that doesn't need an immediate
  response, to reduce coupling and improve resilience to downstream
  failures.
- Synchronous calls between services must have timeouts and a defined
  fallback/circuit-breaker behavior — a hanging downstream call must not
  hang the caller indefinitely.
- Services do not share a database. Cross-service data access goes through
  the owning service's API.

## Idempotency

- Any operation that might be retried (by a client, a queue, or a
  circuit-breaker) must be idempotent, or explicitly documented as unsafe to
  retry.
- Use idempotency keys for state-changing operations exposed over
  unreliable transports.

## Configuration

- Service configuration is externalized (env vars, config service) — not
  hardcoded, and not baked into the build artifact.
