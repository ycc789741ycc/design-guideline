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
- **Security**: no secrets in source control, ever. Least-privilege by
  default. Validate/sanitize all external input. TLS for all external
  traffic. Scan dependencies for vulnerabilities in CI.
- **Testing**: tests run only through `make test-unit` and
  `make test-integration` — never a `run-tests.sh`, a raw
  `pytest`/`jest`/`go test` invocation in docs, or a CI-only command;
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
  smuggled into a test target.
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
