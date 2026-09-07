# Infrastructure as Code

## Principles

- All infrastructure is defined in code (Terraform, Pulumi, CloudFormation,
  or equivalent) and version-controlled — no manual changes made directly
  in a cloud console for anything beyond a documented emergency.
- Emergency manual changes are backfilled into code within a defined window
  (e.g. same business day) so code stays the source of truth.

## Structure

- Environments (dev/staging/prod) are modeled as separate state, sharing
  common modules — not copy-pasted configuration.
- Secrets are never stored in IaC files themselves; they're referenced from
  a secrets manager.
- IaC supplies application configuration by setting the environment
  variables the service already declares in its `.env.example` — it does
  not introduce a parallel, IaC-only set of setting names. See
  [`../shared/configuration.md`](../shared/configuration.md).

## Review

- Infrastructure changes go through the same PR review process as
  application code, with a `plan`/diff output attached so reviewers can see
  the actual effect before it's applied.
- Changes that affect production are applied only after the plan output has
  been reviewed by someone other than the author.
