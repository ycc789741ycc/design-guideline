# Base Guideline

This is the default, overrides-agnostic design guideline. Every
overrides inherits everything here unless it explicitly overrides a file
via `overrides/<org-name>/manifest.yaml` (see the top-level
[README](../README.md) for the precedence rules).

## Contents

| Folder | Scope |
|---|---|
| [`shared/`](shared) | Cross-cutting rules that apply regardless of role: naming, error handling, logging, configuration, build/run entrypoints, security baseline, testing philosophy, version control. Read this first. |
| [`backend/`](backend) | Architecture, API design, data access, and service patterns for backend code. |
| [`frontend/`](frontend) | Component hierarchy, state management, accessibility, and interaction patterns for frontend code. |
| [`infra-ops/`](infra-ops) | Deployment, infrastructure-as-code, monitoring, and incident response for SRE/DevOps. |

## Editing guidance

- Changes to `base/` affect every overrides that has not overridden the
  changed file — treat PRs here as changing the default for the whole company.
- Prefer adding a concrete example (in an `examples/` subfolder) over adding
  another paragraph of prose. Examples are what people and AI tools actually
  follow.
- If a rule only makes sense for one overrides, it doesn't belong in
  `base/` — put it in that overrides' override folder instead.
