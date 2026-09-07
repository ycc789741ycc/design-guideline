# AI Context

Condensed, directive versions of the guideline for AI coding tools (Claude
Code, Cursor, GitHub Copilot, etc.). These are generated, not hand-authored
from scratch — the source of truth is always `base/` (+ an org's overrides
under `overrides/`), never these files.

## Why a separate layer

- AI tools work better with short, directive, example-heavy context than
  with long prose documents meant for human onboarding.
- Keeping generation separate from authoring means `base/` docs stay
  written for humans (with rationale, trade-offs, context) while
  `ai-context/` stays terse and scoped to what an AI tool needs to follow
  the rules correctly.

## Files

- `CLAUDE.md` — root pointer. Place a copy of this (or a symlink) at your
  repo root, or rename to `AGENTS.md`/`.cursorrules` depending on tool.
- `backend-context.md`, `frontend-context.md`, `infra-context.md` —
  per-role condensed context, generated from `base/<role>/` plus the active
  overrides' overrides.

## Regeneration

Regenerate whenever `base/` or an overrides' override files change:

1. Resolve the effective guideline for the target overrides (base +
   overrides, following the precedence rules in
   [`../overrides/README.md`](../overrides/README.md)).
2. Condense each role's resolved docs into directive bullet points and
   keep the concrete code examples verbatim — examples are what AI tools
   pattern-match against most effectively, so don't compress those away.
3. Commit the regenerated `ai-context/` files alongside the guideline
   change in the same PR, so they never drift out of sync with the human
   docs.

This can be done manually for now; if this repo grows multiple
overrides with frequent overrides, consider scripting step 1-2 (e.g. a
small script that reads each `manifest.yaml`, merges files per the `mode`
rules, and re-prompts an LLM to condense the result).
