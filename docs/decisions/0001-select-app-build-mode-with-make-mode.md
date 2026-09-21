# 0001. Select the app's build/run mode with `MODE=dev|prod` on the app targets

- **Status:** Accepted
- **Date:** 2026-09-21
- **Deciders:** Design guideline maintainers

## Context

The eight standard `make` targets gave the app a single way to be built
and run: `build-app` produced one image and `start-app` ran it. Two jobs
pull that one path in opposite directions. Local development wants the
working tree bind-mounted with hot reload, so an edit shows up without a
rebuild. CI and every deployed environment want a minimal,
production-ready image with the code baked in and nothing from the host.

The guideline allowed a hot-reload bind-mount only as an unnamed
"dev compose overlay (`compose.override.yaml`)". That left each repo to
invent its own switch, and the suggested filename is one compose merges
automatically — so any `docker compose up` in that repo, including the
one behind `start-app` in CI, silently picked up the dev mounts.

## Decision

`build-app`, `start-app`, and `stop-app` take `MODE=dev` or `MODE=prod`,
defaulting to `prod`; any other value fails before a recipe runs.

- `MODE=dev` builds the `dev` stage of the repo's single multi-stage
  `Dockerfile` and runs it with `compose.yaml` plus a `compose.dev.yaml`
  overlay that declares the source bind-mounts and the reload command.
- `MODE=prod` builds the `prod` stage — production dependencies only,
  code copied in, non-root — and runs it with `compose.yaml` alone. It is
  the only image scanned, pushed, and promoted; CI and every deployed
  environment use it.
- Tests and gates ignore `MODE` and run in a `test` stage built on
  `prod`, which `build-app` builds in either mode, never bind-mounted.
- The application never reads `MODE`.

The rule itself lives in
[`base/shared/build-and-run.md`](../../base/shared/build-and-run.md#build-and-run-modes).

## Consequences

**Easier**

- One interface for both jobs: developers get hot reload from the same
  targets CI uses, instead of a parallel `dev-*` target set or ad-hoc
  scripts.
- Forgetting the flag is safe — the default is the production artifact,
  so a missing `MODE` can't put a mounted, dev-tooled container into CI
  or a deploy.
- Dev and prod share a base stage, so they can't drift on base image,
  OS packages, or runtime version.

**Harder**

- Every repo now maintains a multi-stage `Dockerfile` with `dev`, `prod`,
  and `test` stages, and a `compose.dev.yaml` — more files and stages than
  a single-image setup.
- `build-app` builds two images per run (the mode's image and `test`),
  so it takes longer; layer caching mitigates but doesn't remove it.
- Tests don't see edits until they are rebuilt into an image: in dev
  mode a developer must re-run `make build-app MODE=dev` before
  `make test-unit`, which is friction a mounted test run wouldn't have.
- Developers who work in dev mode type `MODE=dev` on every app target
  or export it in their shell; it is deliberately not settable from a
  committed file or `.env`.
- Repos that already rely on `compose.override.yaml` must rename it and
  wire it into their Makefile explicitly.

**Accepted risks**

- A dev-stage-only bug (a dependency present in `dev` but missing from
  `prod`) is caught by the `test` stage and CI rather than on the
  developer's machine. Revisit if such misses become common.

## Alternatives considered

- **Separate targets (`start-app-dev`, `build-app-dev`, …)** — doubles
  the app half of the fixed eight-target interface and invites the dev
  variants to drift from the real ones; a variable on the same target is
  the pattern the guideline already uses for narrowing tests (`PATTERN`).
- **Default `MODE=dev`** — friendlier for the most frequent local use,
  but makes the unsafe mode the one you get by omission, including in any
  CI job or script that forgets the flag.
- **Keep `compose.override.yaml`** — compose merges it implicitly, so the
  dev mounts would leak into every prod-mode run that doesn't pass `-f`
  explicitly; opt-in by name is the only reliable separation.
- **Separate `Dockerfile.dev`** — two build definitions drift on base
  image and system packages; stages of one file share them by
  construction.
- **Run tests against the bind-mounted dev container** — faster edit/test
  loop, but tests would exercise the working tree instead of the artifact
  that ships, which the guideline already rejects.
