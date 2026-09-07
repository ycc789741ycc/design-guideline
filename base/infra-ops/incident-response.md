# Incident Response

## During an incident

- Declare the incident explicitly (don't let it stay ambiguous) and assign
  an incident commander to coordinate.
- Communicate status at a known cadence, even if the update is "still
  investigating" — silence during an incident erodes trust more than slow
  progress does.
- Mitigate first, root-cause later — restoring service takes priority over
  fully understanding the failure in the moment.

## Runbooks

- Any alert that has fired more than once has a runbook: symptom →
  diagnostic steps → mitigation steps.
- Runbooks are version-controlled alongside the service they cover and
  updated whenever the mitigation changes.

## After an incident

- Every incident above an agreed severity threshold gets a blameless
  postmortem: timeline, root cause, contributing factors, and concrete
  follow-up actions with owners and dates.
- Postmortems focus on systemic/process fixes, not individual blame — if a
  person "made a mistake," the postmortem should ask why the system made
  that mistake possible.
