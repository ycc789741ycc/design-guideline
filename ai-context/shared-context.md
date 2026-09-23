# Shared Context (condensed from base/shared/)

- **Naming**: kebab-case files, PascalCase classes/types, camelCase
  functions/variables, SCREAMING_SNAKE_CASE constants, snake_case DB
  tables/columns. Booleans read as yes/no questions (`isActive`, not
  `active`). Use terms from the glossary consistently.
- **Errors**: never swallow silently. Use typed/structured errors
  (`Result`/`Either` or custom error classes), not raw strings or bare
  `Error`. Map to stable error codes at API boundaries; never leak
  internal stack traces to clients.
- **Logging**: structured (JSON/key-value), never string concatenation.
  Include `timestamp`, `level`, `service`, `trace_id`. Never log secrets
  or full PII. Levels: ERROR (needs attention), WARN (recovered),
  INFO (state change), DEBUG (verbose, off in prod).
- **Configuration**: every runtime setting comes from an environment
  variable declared in `.env` (git-ignored; `.env.example` committed with
  placeholders). Nothing environment-specific is hardcoded in app code, a
  `Dockerfile`, a `Makefile`, compose, or CI — the only literal allowed is
  a default for an optional, non-sensitive setting (`PORT`, `LOG_LEVEL`,
  timeouts). No defaults for hostnames, URLs, connection strings, or any
  secret — those are required and fail at startup when missing. Read the
  environment once at startup into one validated typed config object; no
  `process.env`/`os.environ` reads elsewhere. Same variable names in every
  environment, only values differ; one artifact promoted unchanged.
- **Containers by default**: infra, the app, both test tiers,
  migrations, the gates (`lint`, `typecheck`, `scan`) and every other
  developer command run inside containers, driven by `make` targets over
  `docker` / `docker compose`. A contributor installs only the container
  runtime and `make` — never a language toolchain, database server,
  linter, scanner, or migration CLI on the host; a bare `npm`/`pip`/`go`/
  `psql`/`alembic` invocation in docs or CI means a target is missing.
  `build-app` produces the image `start-app` runs, and it is the same
  image promoted through environments; migrations and tests run as
  one-off containers from it (`test-unit` with no network, everything
  else on the compose network). Pin image tags (version or digest, never
  `latest`) for infra, base, and tool images; pinned versions live in the
  `Dockerfile`/compose, config arrives at run time via `--env-file .env`.
  Containers run non-root with least privilege; source bind-mounts exist
  only in `MODE=dev`, never in how tests or gates run. Anything
  that genuinely can't be containerized (Xcode builds, native packaging,
  hardware access) keeps the standard target name, states in a comment
  why, and pins/checks the host dependency.
- **Local ports**: every port published on the developer's host (app,
  frontend dev server, any exposed infra) is an uncommon number from
  `10000`–`29999`, never a common default (`3000`, `5000`, `5173`,
  `8000`, `8080`, `8888`, `5432`, `6379`, `27017`, …) or its obvious
  derivative (`18080`, `15432`). Each repo takes one contiguous block
  (e.g. `24810` app, `24811` dev server, `24812` Postgres), exposed as
  optional host-port variables in `.env.example` (`APP_HOST_PORT`)
  defaulting to that block. Conventional ports inside containers and on
  the compose network are fine; infra stays unpublished unless a host
  tool needs it. Deployed environments keep reading `PORT` as assigned.
- **Build/run/test**: every repo exposes eight `make` targets —
  `build-infra`, `build-app`, `start-infra`, `start-app`, `stop-app`,
  `stop-infra`, `test-unit`, `test-integration`. Build never starts,
  start never builds, test neither builds nor starts; app targets never
  start infra. Fixed order: `build-infra → build-app → start-infra →
  [migrate] → start-app`; shutdown is `stop-app` then `stop-infra`.
  Pending migrations run to completion as `start-app`'s first step (a
  `migrate` dependency) — never lazily at first request, never racing
  across replicas; failed migrations fail the start. All targets
  idempotent and fed from `.env`. `stop-infra` preserves data;
  destructive resets get their own named target that is never a
  dependency of a build, start, stop, or test target. Aggregates (`up`,
  `down`, `test`) are allowed only as thin compositions in the correct
  order, never the only way in.
- **Build/run modes**: `build-app`, `start-app`, and `stop-app` take
  `MODE=dev|prod`, default `prod`; any other value fails. `MODE=dev`
  builds the `dev` stage of the one multi-stage `Dockerfile` (dev
  tooling) and runs it with `compose.yaml` + a `compose.dev.yaml`
  overlay that declares the repo bind-mounts and hot-reload command —
  mounts live in compose, never as `-v` in a recipe, and the overlay is
  never named `compose.override.yaml` (compose would auto-merge it into
  prod). `MODE=prod` builds the `prod` stage — production deps only,
  code baked in, non-root — and runs it with `compose.yaml` alone,
  nothing mounted; it is the image scanned, pushed, and promoted, and
  CI and every deployed environment use it. The `dev` image is never
  pushed or deployed. Each mode has its own tag; start never builds and
  fails if that mode's image is missing; both modes migrate first; one
  mode runs at a time and `stop-app` stops either. The app never reads
  `MODE`. Tests and gates ignore `MODE` and run in a `test` stage (prod
  + test deps) that `build-app` builds in either mode, never mounted.
- **Security**: no secrets in source control, ever. Least-privilege by
  default. Validate/sanitize all external input. TLS for all external
  traffic. Scan dependencies and the application image for
  vulnerabilities via `make scan` (containerized, pinned scanner image),
  run in CI as the same target a developer runs locally.
- **Testing**: tests run only through `make test-unit` and
  `make test-integration`, and both execute inside a container built from
  the app image — never a `run-tests.sh`, a raw `pytest`/`jest`/`go test`
  invocation on the host, or a CI-only command;
  narrowing stays a variable on the same target (`make test-unit
  PATTERN=orders`). Tier is decided by what a test *needs*, not by file
  location or speed: `test-unit` is hermetic (no infra, no network, no
  running app) and must pass on a clean checkout; `test-integration`
  crosses real boundaries (datastore, cache, broker, own HTTP surface)
  and assumes infra is already up and migrated — it fails telling the
  caller to run `make start-infra` rather than starting infra itself. It
  cleans up after itself (own schema/namespace, or rollback) so it is
  re-runnable without a destructive reset, and never points at a shared
  or production datastore. Every change includes tests for the new
  behavior in the right tier, and both targets pass with nothing deleted,
  skipped, or weakened to get CI green. Coverage bar scales with risk
  tier — core/stable (auth, payments) needs high coverage + integration
  tests; experimental code can start lighter. Assert on behavior, not
  implementation detail. Lint/typecheck/scan are their own targets, not
  smuggled into a test target, and they run containerized with the tool
  version pinned by the image — never a globally installed linter.
- **Version control**: always fetch and pull the latest upstream (`origin`)
  before modifying a branch, new or existing — never commit on a stale
  base. Resolve conflicts from the pull directly; don't force-push over
  them. Cut every `feature`/`bugfix`/`docs`/`chore`/`refactor`/`test`
  branch from the mainline integration branch (`develop` if the repo has
  one, otherwise `master`/`main`), branching off the freshly fetched
  remote ref. Cut a `hotfix` from the existing released version being
  fixed (that release branch or tag), never from mainline; keep it scoped
  to the defect, and merge it back into mainline and any newer supported
  release line.
- **Worktrees**: cut a new branch as a git worktree
  (`git worktree add ../<repo>-<ticket> -b <branch> origin/develop`) in a
  sibling directory outside the repo, rather than switching branches in
  the shared clone — concurrent agents or people in one checkout lose
  edits, builds, and test runs to the switch. One worktree per task;
  git-ignored setup (`.env`, dependencies, local data) is re-created
  there, not inherited; `git worktree remove` + `prune` once the branch
  is merged or abandoned. Switching in place is only for a checkout that
  is certainly yours alone.
- **Branch naming**: `<change-kind>/<ticket>/<short-description>`, three
  segments in that order (`feature/PROJ-1234/user-export`,
  `bugfix/PROJ-1290/duplicate-invoice-email`,
  `hotfix/PROJ-1188/1.4.2-token-refresh`). Kind is one of `feature`,
  `bugfix`, `hotfix`, `refactor`, `docs`, `chore`, `test`, chosen by what
  the change does — a branch needing two kinds should be split. The ticket
  key is copied verbatim from the tracker, prefix and case included; work
  starts from a ticket, and `no-ticket` is an explained exception, not a
  default. The description is lowercase kebab-case, two to four words,
  contains no slash, and for a hotfix leads with the patched version.
- **Decision records**: a decision that is costly to reverse or non-obvious
  to the next reader (datastore, service boundary, auth/tenancy model,
  public contract, major dependency, an accepted trade-off or deviation
  from this guideline) gets an ADR in `docs/decisions/` at the repo root —
  not `doc/adr/` or `doc/arch/` — named `NNNN-short-title-in-kebab-case.md`,
  four digits from `0001`, numbers never reused or renumbered. A decision
  scoped to one service lives in that repo; one binding several repos lives
  in the guideline repo and also becomes a rule under `base/`. Sections, in
  order: Title (the decision, not the topic), Status + date, Context,
  Decision (active voice, already decided), Consequences (what gets easier
  *and* harder — a record with no costs is unfinished), Alternatives
  considered (each with why it lost). Status is `Proposed`, `Accepted`,
  `Rejected`, `Deprecated`, or `Superseded by NNNN`. An accepted ADR is
  immutable apart from its status line: changed your mind means a new ADR
  superseding the old, never a rewrite, and rejected ADRs are kept. The ADR
  ships in the same PR as the change it justifies, with the
  `docs/decisions/README.md` index updated alongside it; don't write one
  for a reversible or style-level choice.
