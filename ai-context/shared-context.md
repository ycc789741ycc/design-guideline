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
- **Security**: no secrets in source control, ever. Least-privilege by
  default. Validate/sanitize all external input. TLS for all external
  traffic. Scan dependencies for vulnerabilities in CI.
- **Testing**: every change includes tests for the new behavior. Coverage
  bar scales with risk tier — core/stable (auth, payments) needs high
  coverage + integration tests; experimental code can start lighter.
  Assert on behavior, not implementation detail.
