# Backend Guideline

Read [`../shared/`](../shared) first — the rules there (naming, error
handling, logging, security, testing) apply here too and are not repeated.

## Contents

- [`architecture.md`](architecture.md) — layering and module boundaries
- [`api-design.md`](api-design.md) — endpoint naming, versioning, request/response shape
- [`data-access.md`](data-access.md) — query patterns, transactions, migrations
- [`service-patterns.md`](service-patterns.md) — use cases, inter-service communication, idempotency
- [`examples/`](examples) — concrete good/bad code

## Dependency policy

New dependencies require a one-line justification in the PR description:
what problem it solves, and why the standard library or an existing
dependency doesn't already solve it.
