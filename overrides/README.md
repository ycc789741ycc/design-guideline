# Overrides Overrides

This folder lets an overrides or team override parts of the
[base guideline](../base) without forking the whole document set. Each
overrides gets its own subfolder that mirrors `base/`'s structure — but
only contains the files it actually needs to override.

## How to onboard a new overrides

1. Copy the `_template/` folder:
   ```
   cp -r overrides/_template overrides/<org-name>
   ```
2. Fill in `overrides/<org-name>/manifest.yaml` — declare which base
   files you're overriding and in which mode (`replace` or `extend`).
3. Add only the files you listed in the manifest, at the same relative path
   they have under `base/` (e.g. an override of `base/backend/api-design.md`
   goes at `overrides/<org-name>/backend/api-design.md`).
4. Open a PR. Reviewers should check that every file added matches an entry
   declared in `manifest.yaml` — an override file with no manifest entry
   should be treated as a mistake, not merged silently.
5. Regenerate `ai-context/` (see [`../ai-context/README.md`](../ai-context/README.md))
   so AI tools pick up the new resolved rules.

## `manifest.yaml` format

```yaml
overrides: acme
extends: base
overrides:
  - path: shared/security-baseline.md
    mode: replace          # fully replaces the base file
  - path: backend/api-design.md
    mode: extend           # base rules still apply; this file adds to them
notes: >
  Acme requires SOC2-aligned security baseline and uses gRPC
  for all internal service communication.
```

- `mode: replace` — the org file is used instead of the base file entirely.
- `mode: extend` — both are used; the org file should state clearly which
  base section it's adding to or amending, rather than repeating the whole
  base file.

## Resolution walkthrough

Say `org-acme` has this manifest:

```yaml
overrides:
  - path: shared/security-baseline.md
    mode: replace
  - path: backend/api-design.md
    mode: extend
```

When resolving Acme's effective guideline:

| File | Resolved from |
|---|---|
| `shared/naming-conventions.md` | `base/` (no override declared) |
| `shared/security-baseline.md` | `overrides/org-acme/` only (`replace`) |
| `backend/api-design.md` | `base/` **+** `overrides/org-acme/` (`extend`) |
| `frontend/*` | `base/` entirely (Acme has no frontend overrides) |

## When to stop using overrides and fork instead

If an overrides' manifest ends up overriding more than roughly 30% of
base files, the override layer has stopped saving effort — at that point,
consider giving that overrides its own base guideline instead of
continuing to accumulate overrides. This threshold is a prompt to review,
not a hard rule.

## Existing overrides

| Org | Overrides | Notes |
|---|---|---|
| _add rows here as overrides are onboarded_ | | |
