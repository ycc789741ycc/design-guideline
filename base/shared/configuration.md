# Configuration

Every runtime setting a program reads comes from the environment, and the
set of variables a service needs is declared in exactly one place: its
`.env` file. This applies to application code, `Dockerfile`s, `Makefile`s,
compose files, and CI definitions alike — configuration is not something
each of those gets to invent for itself.

Related: [`security-baseline.md`](security-baseline.md) for secret
handling, [`build-and-run.md`](build-and-run.md) for the `make` targets
that pass configuration through,
[`../infra-ops/deployment.md`](../infra-ops/deployment.md) for how values
reach deployed environments.

## The rule

- All configuration is read from environment variables. Nothing reads a
  setting from a hardcoded literal, a checked-in config file, or a
  per-environment code branch.
- The variables a service needs are declared once, in `.env`, and
  documented in a committed `.env.example`.
- No environment-specific value is hardcoded anywhere in the repository —
  not in application code, not in a `Dockerfile`, not in a `Makefile`, not
  in compose or CI files. The only literal allowed at a use site is a
  **default** for a genuinely optional variable (see below).
- `.env` is never committed — it is in `.gitignore` from the first commit.
  `.env.example` is committed and lists every variable with a comment and
  a safe placeholder, never a real value.

## Defaults — the one exception

A default is allowed only when the service is correct and safe running
without the variable set.

- **Defaults are fine for**: things that are the same everywhere and
  harmless if wrong — `PORT`, `LOG_LEVEL`, timeouts, retry counts, page
  sizes, feature flags that default off.
- **Defaults are not allowed for**: anything that differs per environment
  or that identifies, addresses, or authenticates against something
  external — hostnames, URLs, connection strings, queue and bucket names,
  account IDs, credentials, tokens, keys. These have no default; a missing
  value is a startup failure.
- A default must never silently point at a real environment. A default
  that falls back to a production host, or to permissive behavior
  (auth disabled, TLS verification off, debug endpoints on), is a bug even
  though it is "just a default".
- Defaults live next to the variable's declaration in the config module,
  not scattered across every read site. `ENV`/`ARG` in a `Dockerfile` and
  `?=` in a `Makefile` are acceptable for the same category of
  non-sensitive defaults, and must agree with the config module's default
  rather than introducing a second, conflicting one.

## Loading and validation

- Configuration is read **once at startup**, through a single config
  module: read, validate (presence, type, range, enum), and expose a typed
  config object. Everything else takes values from that object.
- Application code never reads `process.env` / `os.environ` /
  `System.getenv` outside that module. One reader means one place to audit
  and one place to see the full contract.
- Missing or invalid required configuration fails at startup, with a
  message naming the variable and what was expected — never a silent
  fallback, and never a failure that surfaces later as a confusing runtime
  error mid-request.
- Config is logged by variable name only, never by value, so a startup log
  can't leak a credential (see
  [`logging-and-observability.md`](logging-and-observability.md)).

## Per-environment values

- The same variable **names** exist in every environment; only the values
  differ. Environment-specific names (`PROD_DB_HOST`) push environment
  logic back into the code.
- **Local / dev**: a developer's own `.env`, copied from `.env.example`.
- **Deployed (dev/staging/prod)**: the platform or orchestrator injects the
  variables, with secret values sourced from the secrets manager. A `.env`
  holding real credentials is never baked into an image, committed, or
  copied to a server by hand.
- Adding a required variable is a breaking change to the deploy contract:
  update `.env.example` and every environment's deploy configuration in
  the same change, or the next deploy fails at startup.

## Containers, make targets, and CI

- A `Dockerfile` builds **one** image that runs in every environment. No
  environment names, hostnames, endpoints, or credentials in the
  Dockerfile or in build args — values arrive at run time. If an image
  can't be promoted from staging to production unchanged, configuration
  has leaked into the build.
- `Makefile` recipes pass configuration through (`--env-file .env`, or by
  exporting the variable) instead of embedding literals. A target that
  needs a value not in `.env` means the variable is missing from
  `.env.example` — not that the recipe is the place to inline it.
- CI reads the same variable names from the CI variable/secret store. CI
  definitions get no hardcoded values either.

## Client-side and build-time configuration

Anything a frontend build inlines into the bundle is public, so the
`.env` rule holds but the classification is stricter.

- Client-exposed variables still come from `.env` (via the framework's
  public prefix — `VITE_`, `NEXT_PUBLIC_`, etc.), never from a literal in
  the source, and never from a hardcoded per-environment branch.
- Only non-secret values may carry a public prefix. A secret behind a
  public prefix is a leak the moment the bundle ships, so API keys and
  tokens stay server-side and are proxied through a backend route.
- Values inlined at build time make the artifact environment-specific,
  which conflicts with promoting one artifact through all environments.
  Prefer serving client configuration at run time (a small
  `/config` endpoint, or values rendered into the page) for anything that
  differs per environment.

## Examples

```dockerfile
# ❌ Bad — environment baked into the image; needs a rebuild per
# environment, and the credential is now in the image history
ENV DATABASE_URL=postgres://svc:hunter2@prod-db.internal:5432/orders
ENV API_BASE_URL=https://api.prod.example.com
```

```dockerfile
# ✅ Good — non-sensitive default only; everything else injected at run time
ENV PORT=8080
ENV LOG_LEVEL=info
# DATABASE_URL, API_BASE_URL: required at run time, no default
```

```makefile
# ❌ Bad — literals in the recipe, duplicated in every target
run:
	DATABASE_URL=postgres://localhost:5432/orders LOG_LEVEL=debug go run ./cmd/api
```

```makefile
# ✅ Good — one declared source; PORT may default, DATABASE_URL may not
PORT ?= 8080

run:
	docker run --env-file .env -p $(PORT):$(PORT) orders-api
```

```typescript
// ❌ Bad — read at the use site, defaulted to a real environment
const client = new ApiClient(
  process.env.API_BASE_URL ?? "https://api.prod.example.com",
);
```

```typescript
// ✅ Good — one validated config module; required values have no default
// config.ts
export const config = loadConfig({
  port: int("PORT", { default: 8080 }),
  logLevel: enum_("LOG_LEVEL", ["debug", "info", "warn", "error"], {
    default: "info",
  }),
  apiBaseUrl: url("API_BASE_URL"), // required — throws at startup if unset
  databaseUrl: url("DATABASE_URL"), // required — throws at startup if unset
});

// api-client.ts
const client = new ApiClient(config.apiBaseUrl);
```
