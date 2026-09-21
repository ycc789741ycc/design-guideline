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
- Never hardcode configuration. Every environment-specific value comes
  from an environment variable declared in `.env` — not from application
  code, a `Dockerfile`, a `Makefile`, compose, or CI. Defaults are allowed
  only for optional, non-sensitive settings, and never for secrets,
  hostnames, URLs, or connection strings.
- Build, run, and test only through the standard `make` targets
  (`build-infra`, `build-app`, `start-infra`, `start-app`, `stop-app`,
  `stop-infra`, `test-unit`, `test-integration`) — keep app and infra
  steps separate, and never let the app start before pending migrations
  have run to completion. `build-app`/`start-app`/`stop-app` take
  `MODE=dev|prod` (default `prod`): `dev` runs a dev-stage image with the
  repo bind-mounted via a `compose.dev.yaml` overlay; `prod` builds and
  runs the production-ready image with nothing mounted, and is the only
  mode CI or any deployed environment uses.
- Run everything in containers by default: infra as compose services,
  the app as the image `build-app` produces, and migrations, both test
  tiers, `lint`, `typecheck`, `scan` and every other developer command as
  containers driven by a `make` target. Never tell someone to install a
  language toolchain, database, linter, scanner, or migration CLI on the
  host, and never put a bare `npm`/`pip`/`go`/`pytest`/`psql` invocation
  in docs, a script, or CI — that means a target is missing. Pin image
  tags (never `latest`). The only exception is a step that genuinely
  can't be containerized (Xcode builds, native packaging, hardware
  access): it keeps the standard target name and says in a comment why,
  with the host dependency pinned and checked.
- Run tests with `make test-unit` (hermetic — nothing running, no
  network) and `make test-integration` (assumes infra is already up and
  migrated), both inside a container built from the app image.
  Never add a parallel way to run tests, never make a test target start
  infra or wipe data, and put each test in the tier matching what it
  actually needs.
- Never swallow errors silently — propagate typed/structured errors.
- New code includes tests appropriate to its risk tier, in the right
  target, with both `make test-unit` and `make test-integration` passing
  and nothing skipped or weakened to get there (see
  `ai-context/shared-context.md`).
- Match existing naming conventions and module boundaries rather than
  introducing new patterns ad hoc.
- Record a decision that is costly to reverse or non-obvious (datastore,
  service boundary, auth/tenancy model, public contract, major dependency,
  an accepted trade-off or deviation from this guideline) as an ADR in
  `docs/decisions/NNNN-short-title-in-kebab-case.md`, in the same PR as the
  change it justifies. Consequences must name what gets harder, and
  alternatives must say why they lost. Once accepted, an ADR is immutable —
  supersede it with a new one rather than rewriting it.
- Always fetch and pull the latest upstream (`origin`) before modifying a
  branch, new or existing — never commit on a stale base.
- Cut new feature, bugfix, docs, chore, refactor, and test branches from
  the mainline integration branch (`develop` if the repo has one,
  otherwise `master`/`main`). Cut a hotfix from the existing released
  version it fixes — that release's branch or tag, never mainline — keep
  it scoped to the defect, and merge it back into mainline afterwards.
- Cut a new branch as a git worktree in a sibling directory
  (`git worktree add ../<repo>-PROJ-1234 -b <branch> origin/develop`),
  not by switching branches in the shared clone — another agent or person
  may be working in that checkout, and a switch yanks the tree out from
  under them. Re-create git-ignored setup (`.env`, dependencies) in the
  new worktree, and `git worktree remove` it once the branch is merged.
- Name every branch `<change-kind>/<ticket>/<short-description>` — e.g.
  `feature/PROJ-1234/user-export`, `bugfix/PROJ-1290/duplicate-invoice-email`,
  `hotfix/PROJ-1188/1.4.2-token-refresh`. Change kind is one of `feature`,
  `bugfix`, `hotfix`, `refactor`, `docs`, `chore`, `test`; the ticket key
  is written exactly as the tracker renders it (`no-ticket` only for the
  rare change with none); the description is lowercase kebab-case, two to
  four words, no slashes. A hotfix leads its description with the patched
  version.
