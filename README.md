# Design Guideline

A single source of truth for how we design and build software — architecture,
code conventions, and operational practices — with a built-in mechanism for
overrides/teams to override the defaults without forking the whole thing.

## Structure

```
/design-guideline
├── base/                 # Default, org-agnostic guideline
│   ├── shared/           # Cross-cutting rules (naming, errors, security, testing...)
│   ├── backend/          # Backend-specific guidance
│   ├── frontend/         # Frontend-specific guidance
│   └── infra-ops/        # SRE / DevOps guidance
│
├── overrides/            # Org-specific overrides on top of base/
│   ├── <org-name>/
│   │   ├── manifest.yaml # Declares what this org overrides and how
│   │   └── ...           # Mirrors base/ structure, only for overridden files
│   └── _template/        # Scaffold for onboarding a new org
│
└── ai-context/           # Condensed, AI-tool-optimized context files
    ├── CLAUDE.md          # Root pointer for AI coding tools (also usable as AGENTS.md)
    └── ...
```

## Resolution / precedence model

1. **Default resolution**: for any given file path (e.g. `shared/security-baseline.md`),
   if the active overrides has a file at that same relative path under
   `overrides/<org>/`, it takes precedence. Otherwise, fall back to `base/`.
2. **`replace` mode**: the org file fully supersedes the base file at that path.
   The base file is ignored entirely once an org's manifest declares `replace`.
3. **`extend` mode**: the org file is read *in addition to* the base file — the
   base rules still apply, the org file adds or amends specific rules on top.
   Org `extend` files should clearly reference which base section they're adding to.
4. **No silent overrides.** Every override, in either mode, must be declared in
   that org's `manifest.yaml`. A file existing on disk under an org folder with
   no manifest entry should be treated as an error / caught in review — this
   keeps every deviation from the default auditable at a glance.
5. **`shared/` overrides are load-bearing decisions.** Prefer overriding
   `backend/`, `frontend/`, or `infra-ops/` before touching `shared/`, since
   shared rules exist specifically to keep behavior consistent across roles.
   An org manifest that overrides a `shared/` file should include a `notes`
   field explaining why.
6. **If an org overrides more than ~30% of base files**, that's a signal the
   org has diverged enough to deserve its own base guideline rather than an
   override layer — revisit at that point instead of continuing to pile on
   overrides.

## How to use this repo

- **Contributing to defaults** → edit files under `base/`. Changes here affect
  every overrides that hasn't explicitly overridden that file.
- **Onboarding a new overrides** → copy `overrides/_template/` to
  `overrides/<org-name>/`, fill in `manifest.yaml`, and add only the files
  you actually need to override.
- **Generating AI context** → `ai-context/` files should be regenerated
  whenever `base/` or an org's overrides change, so AI coding tools always see
  the resolved (base + override) rules rather than stale base-only rules.
  See `ai-context/README.md` for the generation approach.

## Directory-level docs

- [`base/README.md`](base/README.md) — details on the default guideline
- [`overrides/README.md`](overrides/README.md) — how the override
  mechanism works in more depth, with a walkthrough example
- [`ai-context/README.md`](ai-context/README.md) — how AI-facing context files
  are generated and kept in sync
