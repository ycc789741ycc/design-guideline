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

## Containers are the default

Infra, the app, both test tiers, migrations, the quality gates, and
every other command a developer runs against this repo run **in
containers**. The `make` targets are thin wrappers over a container
runtime (`docker` / `docker compose`, or a compatible engine), so a
machine with that runtime and `make` can build, start, test, lint, scan,
and migrate the system without installing a language toolchain, a
database server, or a package manager on the host.

- **Infra runs in containers** — datastores, caches, brokers, and object
  storage come up as compose services with pinned image tags and health
  checks. Not a `brew install postgres` on each developer's machine, and
  not a shared remote instance standing in for local infra.
- **The app runs in a container** — `build-app` produces an image and
  `start-app` runs that image. It is built from the same `Dockerfile` as
  the artifact promoted through environments, so "works locally" and
  "works in staging" are the same statement (see
  [`configuration.md`](configuration.md)).
- **Both test tiers run in containers** — `test-unit` runs inside the app
  image with no services attached and no network; `test-integration` runs
  inside the app image (or a test image built from it) attached to the
  compose network, against the containerized infra. Neither tier depends
  on an interpreter, runtime version, or client library installed on the
  host.
- **Migrations run in the app image** as a one-off container, so the
  migration tool, the driver, and the application agree on version — and
  so the migration that runs locally is byte-for-byte the one the deploy
  runs.
- **The quality gates run in containers too** — `lint`, `typecheck`,
  `scan`, and formatters run inside the app image (or a pinned tool
  image), so a rule that fails in CI fails identically on a laptop, and
  nobody chases a lint error that is really a linter-version difference.
- **So does every other command a developer needs**: code generation,
  dependency installation and lockfile updates, a database console, a
  REPL, one-off maintenance scripts. Each gets a `make` target whose
  recipe runs a container. If the answer to "how do I run this?" is a
  bare `npm`, `pip`, `go`, `psql`, or `alembic` invocation on the host,
  it is missing a target.
- The container runtime and `make` are the only tools a contributor
  installs. A README that opens with "first install Python 3.11,
  Postgres 16, and Redis" is a defect in the repo, not onboarding.
- Pin image tags — a version or digest, never a bare `latest` — for
  infra images, application base images, and tool images alike, so a
  rebuild months later reproduces the same environment. A pinned tool
  version is part of the artifact definition, not configuration: it
  belongs in the `Dockerfile` or compose file, not in `.env`.
- Containers get least privilege like anything else: a non-root user,
  only the ports, mounts, and capabilities they need (see
  [`security-baseline.md`](security-baseline.md)).
- Bind-mounting the working tree for hot reload is a local-development
  convenience that lives in a dev compose overlay
  (`compose.override.yaml`), never in the image and never in how the test
  targets run. A test that passes only with the host's source mounted is
  not testing the artifact.

### When something genuinely can't be containerized

Some steps can't run in a container — an iOS/macOS build that needs
Xcode, native desktop packaging, hardware or GPU access the runtime
doesn't expose, or a frontend dev server developers expect to run
natively. That is an exception, declared as one:

- The target keeps its standard name and contract; only its recipe runs
  on the host.
- The recipe carries a comment saying why it can't be containerized and
  what the host must have installed.
- The host dependency is pinned and checked — a version file plus an
  early, explicit failure — never assumed.
- Everything that *can* still run in a container does. A natively-run
  frontend dev server doesn't make the backend's tests native too.

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
- Every recipe drives the container runtime by default: `build-infra`
  pulls/builds images, `build-app` builds the app image, the `start-`
  and `stop-` targets manage compose services, and the test targets run
  inside a container (see
  [Containers are the default](#containers-are-the-default)).

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
- `migrate` runs as a one-off container from the app image, attached to
  the infra network — not a migration CLI installed on the developer's
  host. That keeps the runner, the driver, and the schema history the
  same locally, in CI, and in the deploy job.
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
- Both targets run the suite **inside a container** built from the app
  image, not against a host-installed toolchain: `test-unit` with no
  network and no services attached, `test-integration` on the compose
  network next to the infra containers. The same command therefore
  behaves identically on a developer's laptop and on a CI runner, which
  needs nothing installed but the container runtime and `make`.
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
  target, so a failure names which gate failed. Like the test targets,
  they run in a container with the tool and its version pinned by the
  image — never a globally installed linter whose version differs per
  machine — and they need nothing running, so they behave like
  `test-unit` with respect to infra.

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

Every recipe runs through the container runtime — nothing here assumes a
language toolchain, database client, or test runner installed on the host.

```makefile
.PHONY: build-infra build-app start-infra start-app stop-app stop-infra \
        test-unit test-integration test lint typecheck scan deps \
        db-console migrate up down reset-infra

# --- build ---------------------------------------------------------------
build-infra:                      # pull/build backing services, validate IaC
	docker compose --env-file .env pull
	terraform -chdir=infra validate

build-app:                        # compile/bundle our code into an artifact
	docker build -t orders-api:local .

# --- run -----------------------------------------------------------------
start-infra:                      # start backing services, wait for health
	docker compose --env-file .env up -d --wait postgres redis

migrate:                          # one-off container on the infra network;
                                  # assumes infra is up
	docker compose --env-file .env run --rm --no-deps api ./bin/migrate up

start-app: migrate                # migrations first, then serve
	docker compose --env-file .env up -d api

# --- stop ----------------------------------------------------------------
stop-app:
	docker compose stop api

stop-infra:                       # stops services; volumes are preserved
	docker compose stop postgres redis

# --- test ----------------------------------------------------------------
test-unit:                        # hermetic; in a container, no network
	docker run --rm --network none --env-file .env \
		orders-api:local pytest tests/unit $(if $(PATTERN),-k $(PATTERN),)

test-integration:                 # container on the infra network; assumes
                                  # infra is up and migrated
	docker compose --env-file .env run --rm --no-deps api \
		pytest tests/integration $(if $(PATTERN),-k $(PATTERN),)

test: test-unit test-integration  # thin composition, in that order

# --- gates: own targets, containerized, nothing running ------------------
lint:
	docker run --rm --network none orders-api:local ruff check .

typecheck:
	docker run --rm --network none orders-api:local mypy src

scan:                             # pinned tool image, not a host install
	docker run --rm -v $(PWD):/src:ro aquasec/trivy:0.54.1 fs /src

# --- developer tooling: a target per command, never a host invocation ----
deps:                             # resolve/update the lockfile in-image
	docker run --rm -v $(PWD):/app orders-api:local pip-compile requirements.in

db-console:                       # assumes infra is up
	docker compose --env-file .env exec postgres \
		sh -c 'psql -U "$$POSTGRES_USER" "$$POSTGRES_DB"'


# --- convenience (thin composition, correct order) -----------------------
up: build-infra build-app start-infra start-app
down: stop-app stop-infra

# --- destructive: explicit, never a dependency ---------------------------
reset-infra:                      # DESTROYS local database and cache data
	docker compose down -v
```
