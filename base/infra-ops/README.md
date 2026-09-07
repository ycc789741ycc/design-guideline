# Infra-Ops Guideline (SRE + DevOps)

Read [`../shared/`](../shared) first. This folder combines SRE and DevOps
concerns since they're commonly the same team or tightly coupled — split
into separate folders under an overrides override if your org has
genuinely separate teams with separate concerns.

## Contents

- [`deployment.md`](deployment.md) — environments, CI/CD, rollback, feature flags
- [`infrastructure-as-code.md`](infrastructure-as-code.md) — IaC structure and review
- [`monitoring-and-alerting.md`](monitoring-and-alerting.md) — what to monitor, alerting philosophy, on-call
- [`incident-response.md`](incident-response.md) — during/after an incident, runbooks, postmortems
- [`capacity-and-scaling.md`](capacity-and-scaling.md) — scaling principles and practices
