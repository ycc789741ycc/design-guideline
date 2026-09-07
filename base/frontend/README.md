# Frontend Guideline

Read [`../shared/`](../shared) first — the rules there (naming, error
handling, logging, security, testing) apply here too and are not repeated.

## Contents

- [`component-hierarchy.md`](component-hierarchy.md) — atomic → composite → feature → page
- [`state-management.md`](state-management.md) — local vs. server vs. global state
- [`accessibility.md`](accessibility.md) — baseline a11y requirements
- [`interaction-patterns.md`](interaction-patterns.md) — loading, error, empty states, navigation
- [`examples/`](examples) — concrete good/bad code

An overrides with its own design system or branding should override
`visual-foundations.md` (color, typography, spacing) under its
`overrides/<org>/frontend/` folder — this base guideline intentionally
does not prescribe a visual language, only structural and behavioral rules.
