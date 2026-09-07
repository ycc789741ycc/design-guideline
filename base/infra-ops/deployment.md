# Deployment

## Environments

- Standard promotion path: `dev → staging → production`. No environment is
  skipped for production-bound changes.
- Environments are configured identically apart from scale and secrets —
  configuration drift between staging and production is treated as a bug.

## CI/CD

- Every change to a shared branch runs the full test suite, linter, and
  security scan before merge is allowed.
- Deployments are automated (no manual server access to deploy) and
  triggered from a single, auditable pipeline.
- Deployments are small and frequent rather than large and infrequent —
  smaller changes are easier to attribute if something breaks.

## Rollback

- Every deployment can be rolled back to the previous known-good version
  without a manual, bespoke procedure — rollback is a tested, first-class
  pipeline action, not an emergency improvisation.
- Database migrations that accompany a deploy are written to be
  backward-compatible with the previous version of the application code,
  so a rollback doesn't require an immediate matching migration rollback.

## Feature flags

- Risky or incomplete changes ship behind a flag, decoupling deploy from
  release.
- Flags have an owner and an expected removal date — flags are not meant
  to live forever.
