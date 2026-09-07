# Build and Run

Every repository exposes the same `make` interface, split along two axes:
**build vs. run** (producing an artifact vs. starting something), and
**app vs. infra** (our code vs. the backing services it depends on).

This is the contract for local development, CI, and deployment alike —
one definition of how the system builds and starts, not a Makefile for
humans and a separate pile of shell scripts for the pipeline.

Related: [`configuration.md`](configuration.md) (make targets read
configuration from `.env`, never from literals),
[`../infra-ops/deployment.md`](../infra-ops/deployment.md) (how the
pipeline invokes these targets).

## The six targets

| Target | Responsibility |
|---|---|
| `build-infra` | Produce/prepare everything the backing services need — pull or build their images, render IaC, validate infra definitions. No application code. |
| `build-app` | Produce the application artifact — compile, bundle, build the app image. No infrastructure, no running services. |
| `start-infra` | Start the backing services (database, cache, broker, object storage, network) and wait until they are **healthy**. |
| `start-app` | Run pending migrations, then start the application. Assumes infra is already up. |
| `stop-app` | Stop the application. Leaves infra running. |
| `stop-infra` | Stop the backing services. Preserves their data. |

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
```

Shutdown is the reverse: `stop-app` before `stop-infra`. Stopping infra
out from under a running app produces connection errors that look like
application bugs.

- `start-app` **must not** start infra, and `start-infra` must not start
  the app. Keeping them separate is what makes it possible to restart the
  app against a warm database, or to point the app at already-running
  shared infra.
- Aggregate convenience targets (`make up`, `make down`) are allowed as
  thin compositions of the six, in the correct order. They must not
  become the only way to start the system, and must not reimplement the
  work themselves.
- Build targets do not start anything. Start targets do not build —
  a start target that silently rebuilds hides how stale the artifact is.
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

## Destructive operations are separate and named

- `stop-infra` stops services; it does not delete volumes, drop
  databases, or discard state. Losing local data must never be the
  side effect of stopping something.
- Anything destructive gets its own explicitly named target
  (`reset-infra`, `clean`) that states what it destroys, and it is never
  a dependency of a build, start, or stop target.

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
        migrate up down reset-infra

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

# --- convenience (thin composition, correct order) -----------------------
up: build-infra build-app start-infra start-app
down: stop-app stop-infra

# --- destructive: explicit, never a dependency ---------------------------
reset-infra:                      # DESTROYS local database and cache data
	docker compose down -v
```
