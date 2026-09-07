# Security Baseline

This is the minimum bar for all code, regardless of overrides. Orgs with
stricter compliance requirements (SOC2, HIPAA, PCI, etc.) should override
this file under `overrides/<org>/shared/security-baseline.md` — see
[`overrides/README.md`](../../overrides/README.md).

## Secrets

- No secrets, credentials, or API keys committed to source control, ever —
  including in comments, test fixtures, or commit history.
- Secrets are loaded from a secrets manager or environment variables at
  runtime, never hardcoded.

## Auth

- Authentication and authorization logic lives in one well-known place per
  service — not duplicated ad hoc across endpoints.
- Default to least privilege: new roles/permissions start with no access
  and are granted explicitly.
- All external-facing endpoints require explicit authentication unless
  deliberately public (and documented as such).

## Data handling

- PII is identified and classified; access to it is logged.
- All external network traffic uses TLS.
- Input from any external source (user input, third-party API responses,
  file uploads) is validated and sanitized before use — never trusted by
  default.

## Dependencies

- Dependencies are scanned for known vulnerabilities as part of CI.
- New dependencies require a stated reason before being added (see
  [`backend/README.md`](../backend/README.md) / equivalent frontend policy).
