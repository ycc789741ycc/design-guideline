# Design Guideline — AI Context Root

This project follows the design guideline in this repository. Read the
relevant condensed context file(s) below before writing or reviewing code,
in addition to any human-facing docs under `base/` or `overrides/` if
more detail is needed.

- Working on backend code → read `ai-context/backend-context.md`
- Working on frontend code → read `ai-context/frontend-context.md`
- Working on infra/deployment/CI → read `ai-context/infra-context.md`
- All of the above inherit rules from `ai-context/shared-context.md` —
  read this regardless of which role you're working in.

## Resolution note

If this repository belongs to a specific overrides with overrides
(see `overrides/<org-name>/manifest.yaml`), the context files below
already reflect that overrides' resolved rules (base + overrides).
Do not re-derive rules from `base/` alone if an override exists — the
`ai-context/` files are the authoritative, already-merged version.

## Non-negotiables (apply regardless of role)

- Never commit secrets, credentials, or API keys.
- Never swallow errors silently — propagate typed/structured errors.
- New code includes tests appropriate to its risk tier (see
  `ai-context/shared-context.md`).
- Match existing naming conventions and module boundaries rather than
  introducing new patterns ad hoc.
