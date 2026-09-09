# Build, Run, and Test

Every repository exposes the same `make` interface, split along two axes:
**build vs. run vs. test** (producing an artifact, starting something, or
verifying it), and **app vs. infra** (our code vs. the backing services it
depends on).

This is the contract for local development, CI, and deployment alike —
one definition of how the system builds, starts, and is tested, not a
Makefile for humans and a separate pile of shell scripts for the pipeline.

Related: [`configuration.md`](configuration.md) (make targets read
configuration from `.env`, never from literals),
[`testing-philosophy.md`](testing-philosophy.md) (what belongs in each
test tier), [`../infra-ops/deployment.md`](../infra-ops/deployment.md)
(how the pipeline invokes these targets).

## The eight targets

| Target | Responsibility |
|---|---|
| `build-infra` | Produce/prepare everything the backing services need — pull or build their images, render IaC, validate infra definitions. No application code. |
| `build-app` | Produce the application artifact — compile, bundle, build the app image. No infrastructure, no running services. |
| `start-infra` | Start the backing services (database, cache, broker, object storage, network) and wait until they are **healthy**. |
| `start-app` | Run pending migrations, then start the application. Assumes infra is already up. |
| `stop-app` | Stop the application. Leaves infra running. |
| `stop-infra` | Stop the backing services. Preserves their data. |
| `test-unit` | Run the tests that need nothing running — no infra, no app, no network. |
| `test-integration` | Run the tests that cross a real boundary (DB, cache, broker, HTTP surface). Assumes infra is already up. |

- **"Infra"** means the runtime our code depends on but does not itself
  contain: datastores, caches, brokers, object storage, and the network
  they share. **"App"** means our own service(s).
- Nothing outside this table is required, but nothing in it is optional:
  a repo that can't provide a target still declares it, with a recipe
  that explains why it's a no-op rather than leaving the name undefined.

## Ordering

The dependency order is fixed, and each target does only its own step:

```
build-infra → build-app → start-infra → [migrate] → start-app

test-unit          (no dependencies — runs on its own)
test-integration   (after start-infra, and after [migrate])
```

Shutdown is the reverse: `stop-app` before `stop-infra`. Stopping infra
out from under a running app produces connection errors that look like
application bugs.

- `start-app` **must not** start infra, and `start-infra` must not start
  the app. Keeping them separate is what makes it possible to restart the
  app against a warm database, or to point the app at already-running
  shared infra.
- Aggregate convenience targets (`make up`, `make down`) are allowed as
  thin compositions of these targets, in the correct order. They must not
  become the only way to start the system, and must not reimplement the
  work themselves.
- Build targets do not start anything. Start targets do not build —
  a start target that silently rebuilds hides how stale the artifact is.
- Test targets do not build or start anything either — see
  [Tests run through make](#tests-run-through-make).
- All targets are idempotent: safe to re-run when already built,
  already running, or already stopped.

## Migrations run before the app starts

- Pending migrations run to completion **before** any application process
  begins serving. This is `start-app`'s first step, expressed as a
  dependency on a `migrate` target — not a manual step in a runbook, and
  not something the app does lazily on its first request.
- If migrations fail, `start-app` fails and the application does not
  start. Booting against a half-migrated schema is worse than not
  booting at all.
- `migrate` is also a standalone target, so it can be run and inspected
  on its own. It assumes infra is up rather than depending on
  `start-infra` — otherwise `start-app` would start infra transitively,
  which is exactly the coupling these targets exist to avoid.
- Migrations need infra to be up and healthy, which is why `start-infra`
  precedes them — not because the app needs a warm-up.
- Exactly one migration runner executes at a time. When the app runs as
  multiple replicas, migrations are a separate step in the deploy (a job
  or pre-deploy phase), never a race between replicas at boot.
- Because migrations run before the new code does, they must be
  backward-compatible with the currently deployed version — see
  [`../infra-ops/deployment.md`](../infra-ops/deployment.md) (rollback)
  and [`../backend/data-access.md`](../backend/data-access.md)
  (migration conventions).

## Tests run through make

Tests are part of the same interface, not a separate convention. Nobody
needs to know whether this repo uses pytest, jest, or `go test` to run
its suite, and CI invokes exactly what a developer does locally.

- Two targets, split by what they need to run: `test-unit` needs nothing
  running; `test-integration` needs infra up. The split is the contract —
  a single `make test` that hides the distinction forces every caller to
  pay the infra cost, and makes a failure ambiguous between "our logic is
  wrong" and "the database wasn't ready".
- `test-unit` is hermetic: no database, no broker, no network, no
  filesystem outside a temp directory, no dependency on a running app.
  It must pass on a clean checkout with nothing else started, which is
  what makes it the fast pre-commit and first-CI-stage check.
- `test-integration` exercises real boundaries — the actual datastore,
  cache, broker, and the app's own HTTP surface. It assumes infra is up
  and migrated, for the same reason `migrate` does: a test target that
  starts infra transitively re-couples exactly what these targets keep
  apart. When infra isn't up, it fails with a message saying to run
  `make start-infra` — it does not silently start it.
- Both targets take configuration from `.env` like every other target,
  pointed at a local or ephemeral test instance. Neither ever runs
  against a shared or production datastore, and neither needs
  credentials that aren't already declared in `.env.example`.
- `test-integration` leaves infra in a reusable state: it creates its own
  schema/namespace/prefix or rolls back what it wrote, so it is
  idempotent and re-runnable without a `reset-infra` in between.
- An aggregate `test` target is allowed as a thin composition
  (`test: test-unit test-integration`), in that order, and must not be
  the only way to run either half.
- Both are runnable in isolation and both report a non-zero exit status
  on failure — CI gating depends on it. Narrowing to a subset stays a
  variable on the same target (`make test-unit PATTERN=orders`), not a
  new target per directory.
- Lint, type-check, and security-scan steps get their own targets
  (`lint`, `typecheck`, `scan`) rather than being smuggled into a test
  target, so a failure names which gate failed.

## Destructive operations are separate and named

- `stop-infra` stops services; it does not delete volumes, drop
  databases, or discard state. Losing local data must never be the
  side effect of stopping something.
- Anything destructive gets its own explicitly named target
  (`reset-infra`, `clean`) that states what it destroys, and it is never
  a dependency of a build, start, stop, or test target — a test suite
  that wipes the database to get a clean slate takes the developer's
  local data with it.

## Configuration

- Targets take configuration from `.env` — passed through with
  `--env-file .env` or by exporting variables — never from literals in
  the recipe (see [`configuration.md`](configuration.md)).
- The same target names run in CI and in the deploy pipeline, with
  different values in the environment. If the pipeline needs a step the
  Makefile doesn't have, add it to the Makefile rather than growing a
  parallel implementation.

## Example

```makefile
.PHONY: build-infra build-app start-infra start-app stop-app stop-infra \
        test-unit test-integration test migrate up down reset-infra

# --- build ---------------------------------------------------------------
build-infra:                      # pull/build backing services, validate IaC
	docker compose --env-file .env pull
	terraform -chdir=infra validate

build-app:                        # compile/bundle our code into an artifact
	docker build -t orders-api:local .

# --- run -----------------------------------------------------------------
start-infra:                      # start backing services, wait for health
	docker compose --env-file .env up -d --wait postgres redis

migrate:                          # standalone; assumes infra is up
	docker run --rm --env-file .env orders-api:local ./bin/migrate up

start-app: migrate                # migrations first, then serve
	docker compose --env-file .env up -d api

# --- stop ----------------------------------------------------------------
stop-app:
	docker compose stop api

stop-infra:                       # stops services; volumes are preserved
	docker compose stop postgres redis

# --- test ----------------------------------------------------------------
test-unit:                        # hermetic; needs nothing running
	docker run --rm --env-file .env orders-api:local pytest tests/unit

test-integration:                 # assumes infra is up and migrated
	docker compose --env-file .env run --rm api pytest tests/integration

test: test-unit test-integration  # thin composition, in that order

# --- convenience (thin composition, correct order) -----------------------
up: build-infra build-app start-infra start-app
down: stop-app stop-infra

# --- destructive: explicit, never a dependency ---------------------------
reset-infra:                      # DESTROYS local database and cache data
	docker compose down -v
```
