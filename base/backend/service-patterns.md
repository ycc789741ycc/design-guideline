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

- Service configuration is externalized as environment variables declared
  in `.env` — not hardcoded, and not baked into the build artifact.
- A service reads its environment once at startup into a single validated,
  typed config object; the rest of the service takes values from that
  object rather than reading the environment directly.
- Required configuration (anything environment-specific, or any secret)
  has no default and fails the service at startup when missing.
- Full rules, including what may carry a default:
  [`../shared/configuration.md`](../shared/configuration.md).
