# Infra-Ops Context (condensed from base/infra-ops/)

Read `shared-context.md` first — applies here too.

- **Deployment**: dev → staging → production, no skipped environments.
  `make test-unit` + `make test-integration` + lint + security scan gate
  every merge, invoked as those same targets rather than CI-only
  commands (`test-unit` first; `test-integration` after the pipeline
  starts infra and migrates, against infra it created for that run). Deploys are
  small/frequent, automated, and always rollback-able without a bespoke
  procedure. DB migrations must be backward-compatible with the previous
  app version. Every environment supplies the same variable names (from
  the service's `.env.example`) with different values; one build artifact
  is promoted through all environments unchanged. Adding a required
  variable means updating `.env.example` and every environment's config in
  the same change. The pipeline builds and starts via the standard `make`
  targets (`build-infra`, `build-app`, `start-infra`, `start-app`,
  `stop-app`, `stop-infra`, `test-unit`, `test-integration`), not
  pipeline-only scripts, with infra before app; migrations are their own
  step, completed before any new instance serves traffic.
- **Infrastructure as code**: all infra defined in code and
  version-controlled; no manual console changes except documented
  emergencies, backfilled into code same-day. Secrets never live in IaC
  files — reference a secrets manager. IaC supplies app config by setting
  the env var names the service already declares, not a parallel set of
  IaC-only names. No hostnames, endpoints, credentials, or environment
  names hardcoded in `Dockerfile`s, `Makefile`s, or pipeline definitions —
  only non-sensitive defaults (`PORT`, `LOG_LEVEL`).
- **Monitoring/alerting**: track golden signals (latency, traffic, errors,
  saturation) per service. Every alert must be actionable — page on
  symptoms/user impact, not on every underlying cause. Every page links
  to a runbook.
- **Incident response**: declare incidents explicitly, assign an incident
  commander, mitigate before root-causing. Every incident above the
  severity threshold gets a blameless postmortem with owned follow-ups.
- **Capacity/scaling**: scale from measured load, not guesswork. Prefer
  horizontal scaling of stateless components. Test auto-scaling policies,
  don't just configure them.
