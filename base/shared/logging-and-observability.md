# Logging & Observability

## Logging

- Use structured logging (JSON or key-value) — never plain string
  concatenation — so logs are queryable in aggregation tools.
- Standard fields on every log line: `timestamp`, `level`, `service`,
  `trace_id` (or `request_id`), `message`.
- Log levels:
  - `ERROR` — something failed and needs human attention.
  - `WARN` — something unexpected happened but the system recovered.
  - `INFO` — significant state changes (request received, job completed).
  - `DEBUG` — verbose detail, off by default in production.
- Never log secrets, tokens, passwords, or full PII payloads. Redact or
  hash sensitive fields before logging.

## Observability

- Every service exposes health/readiness endpoints.
- Every external call (DB, downstream service, third-party API) is
  instrumented with latency and error-rate metrics.
- Propagate a trace ID across service boundaries so a single request can be
  followed end-to-end.
- Alerts should be actionable — if an alert fires and the response is
  "nothing to do," the threshold or the alert itself needs fixing (see
  [`infra-ops/monitoring-and-alerting.md`](../infra-ops/monitoring-and-alerting.md)).
