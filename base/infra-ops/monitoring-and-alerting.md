# Monitoring & Alerting

## What gets monitored

- Golden signals per service: latency, traffic, errors, saturation.
- Business-relevant metrics where they exist (e.g. checkout completion
  rate) in addition to system metrics.

## Alerting philosophy

- Every alert must be actionable — if the response to an alert firing is
  "nothing to do," fix the threshold or delete the alert.
- Alerts are tied to user/business impact where possible (e.g. "error rate
  above X% for Y minutes"), not just raw resource thresholds, to avoid
  noisy pages for things that don't actually matter yet.
- Page on symptoms (users are affected), not on every underlying cause —
  route non-urgent causes to a ticket/dashboard instead of a page.

## On-call

- On-call rotation and escalation path are documented and discoverable —
  not tribal knowledge.
- Every page has a linked runbook (see
  [`incident-response.md`](incident-response.md)) or, if none exists yet,
  creating one is a required follow-up from the incident.
