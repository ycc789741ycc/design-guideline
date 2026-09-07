# Infra-Ops Context (condensed from base/infra-ops/)

Read `shared-context.md` first — applies here too.

- **Deployment**: dev → staging → production, no skipped environments.
  Full test suite + lint + security scan gate every merge. Deploys are
  small/frequent, automated, and always rollback-able without a bespoke
  procedure. DB migrations must be backward-compatible with the previous
  app version.
- **Infrastructure as code**: all infra defined in code and
  version-controlled; no manual console changes except documented
  emergencies, backfilled into code same-day. Secrets never live in IaC
  files — reference a secrets manager.
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
