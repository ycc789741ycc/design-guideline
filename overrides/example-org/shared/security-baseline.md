# Security Baseline (example-org override — replace)

This fully replaces [`base/shared/security-baseline.md`](../../../base/shared/security-baseline.md)
for example-org. Everything in the base file is either restated or
superseded here — the base file is not consulted once this override is
active.

## Additional requirements beyond base

- SOC2 Type II controls apply: all production access is logged and
  reviewed quarterly.
- PII fields are encrypted at rest (base only requires TLS in transit).
- Access to customer data requires a documented business justification,
  not just a role grant.
- Third-party vendors touching customer data must have a signed DPA on
  file before integration.

## Retained from base (restated for completeness)

- No secrets committed to source control. Secrets reach the process as
  environment variables declared in `.env` (git-ignored; only
  `.env.example` with placeholders is committed), never hardcoded in
  application code, a `Dockerfile`, a `Makefile`, or CI, and never given a
  default value.
- Least-privilege default for new roles/permissions.
- Input from external sources is validated and sanitized before use.
- Dependencies are scanned for known vulnerabilities in CI.
