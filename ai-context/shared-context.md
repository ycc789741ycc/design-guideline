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
- **Build/run**: every repo exposes six `make` targets — `build-infra`,
  `build-app`, `start-infra`, `start-app`, `stop-app`, `stop-infra`.
  Build never starts, start never builds; app targets never start infra.
  Fixed order: `build-infra → build-app → start-infra → [migrate] →
  start-app`; shutdown is `stop-app` then `stop-infra`. Pending
  migrations run to completion as `start-app`'s first step (a `migrate`
  dependency) — never lazily at first request, never racing across
  replicas; failed migrations fail the start. All targets idempotent and
  fed from `.env`. `stop-infra` preserves data; destructive resets get
  their own named target that is never a dependency.
- **Security**: no secrets in source control, ever. Least-privilege by
  default. Validate/sanitize all external input. TLS for all external
  traffic. Scan dependencies for vulnerabilities in CI.
- **Testing**: every change includes tests for the new behavior. Coverage
  bar scales with risk tier — core/stable (auth, payments) needs high
  coverage + integration tests; experimental code can start lighter.
  Assert on behavior, not implementation detail.
