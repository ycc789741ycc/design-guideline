# Deployment

## Environments

- Standard promotion path: `dev → staging → production`. No environment is
  skipped for production-bound changes.
- Environments are configured identically apart from scale and secrets —
  configuration drift between staging and production is treated as a bug.
- Every environment supplies the *same* variable names, declared in the
  service's `.env.example`; only the values differ. Nothing
  environment-specific is hardcoded in the application, `Dockerfile`,
  `Makefile`, or pipeline definition — see
  [`../shared/configuration.md`](../shared/configuration.md).
- One build artifact is promoted through all environments unchanged. If an
  image has to be rebuilt per environment, configuration has leaked into
  the build.
- Adding a required variable is a deploy-contract change: land it in
  `.env.example` and in every environment's configuration in the same
  change, before the deploy that needs it.

## CI/CD

- Every change to a shared branch runs the full test suite, linter, and
  security scan before merge is allowed.
- Deployments are automated (no manual server access to deploy) and
  triggered from a single, auditable pipeline.
- The pipeline builds and starts the system through the standard `make`
  targets (`build-infra`, `build-app`, `start-infra`, `start-app`, and
  their `stop-` counterparts) rather than a parallel set of pipeline-only
  scripts — see [`../shared/build-and-run.md`](../shared/build-and-run.md).
  Infra and application steps stay separate, in that order.
- Pending migrations run to completion, as their own step, before any new
  application instance serves traffic. Never at first request, and never
  as a race between replicas at boot.
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
