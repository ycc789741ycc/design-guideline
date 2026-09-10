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
  have run to completion.
- Run tests with `make test-unit` (hermetic — nothing running) and
  `make test-integration` (assumes infra is already up and migrated).
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
- Always fetch and pull the latest upstream (`origin`) before modifying a
  branch, new or existing — never commit on a stale base.
- Cut new feature, bugfix, docs, chore, refactor, and test branches from
  the mainline integration branch (`develop` if the repo has one,
  otherwise `master`/`main`). Cut a hotfix from the existing released
  version it fixes — that release's branch or tag, never mainline — keep
  it scoped to the defect, and merge it back into mainline afterwards.
- Name every branch `<change-kind>/<ticket>/<short-description>` — e.g.
  `feature/PROJ-1234/user-export`, `bugfix/PROJ-1290/duplicate-invoice-email`,
  `hotfix/PROJ-1188/1.4.2-token-refresh`. Change kind is one of `feature`,
  `bugfix`, `hotfix`, `refactor`, `docs`, `chore`, `test`; the ticket key
  is written exactly as the tracker renders it (`no-ticket` only for the
  rare change with none); the description is lowercase kebab-case, two to
  four words, no slashes. A hotfix leads its description with the patched
  version.
