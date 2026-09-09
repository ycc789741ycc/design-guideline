# Testing Philosophy

Related: [`build-and-run.md`](build-and-run.md) for the `make` interface
these tests run behind.

## Tests run through `make`, in two tiers

Every repository runs its tests through two `make` targets, and only
those two. They are part of the standard `make` interface described in
[`build-and-run.md`](build-and-run.md) — the same commands a developer
types locally are the ones CI invokes, so a green pipeline and a green
checkout mean the same thing.

| Target | Needs | Contains |
|---|---|---|
| `make test-unit` | Nothing running | Pure logic: domain rules, calculations, validation, mapping, error paths. Dependencies are substituted at the boundary. |
| `make test-integration` | Infra up and migrated | Anything crossing a real boundary: repository/query code against the real datastore, cache and broker behavior, the app's own HTTP surface, migrations applying cleanly. |

- Which tier a test belongs to is decided by what it *needs*, not by
  where its file lives or how slow it is. A "unit" test that reaches a
  database is an integration test in the wrong target, and it will fail
  the moment someone runs `make test-unit` on a clean checkout.
- `test-unit` is the fast gate: hermetic, no infra, no network, safe to
  run on every save and as the first CI stage. `test-integration` is the
  slower gate that catches what mocks cannot — SQL that doesn't compile,
  a constraint that rejects the write, a serialization mismatch.
- Neither target starts infra for you, and neither is destructive; see
  [`build-and-run.md`](build-and-run.md) for why, and for the invocation
  and cleanup rules.
- Never run a test suite against a shared or production datastore.
  Integration tests point at a local or ephemeral instance, configured
  from `.env` like everything else.
- Test invocation is not a place for a parallel convention: no
  `run-tests.sh`, no README paragraph of raw `pytest`/`jest`/`go test`
  incantations, no CI-only test command. If a suite needs a new way to be
  run, it becomes a variable on the existing target.

## What "done" means

A change is not done until it has:
- Tests covering the new behavior, in the right tier: `test-unit` at
  minimum, plus `test-integration` where the change crosses a boundary —
  DB, network, another service.
- Both `make test-unit` and `make test-integration` passing, with no
  tests deleted, skipped, or weakened just to make CI green.
- Tests that assert on *behavior*, not implementation detail (avoid
  brittle tests that break on harmless refactors).

## Coverage expectations by risk tier

| Tier | Examples | Expectation |
|---|---|---|
| Core / stable | auth, payments, data models | High coverage, `test-integration` coverage required, strict review |
| Standard features | most application logic | `test-unit` required, `test-integration` where boundaries are crossed |
| Experimental / volatile | feature flags, A/B variants | Lighter — smoke tests acceptable, can tighten once it stabilizes |

## Why this matters more with AI-assisted development

When AI tools generate a meaningful share of code, tests become the primary
mechanism for verifying correctness at scale — a human reviewing a large
AI-generated diff line-by-line doesn't scale, but a human confirming the
tests actually assert the right behavior does. Treat test quality as a
first-class review criterion, not an afterthought.

## Anti-patterns

- Tests that just re-implement the function under test (assert-nothing
  tests).
- Overuse of mocks to the point that the test no longer exercises real
  integration points that matter.
- Snapshot tests with no human ever reviewing what the snapshot actually
  contains.
- Integration tests parked in `test-unit`, so the fast gate needs a
  database — or unit-level logic parked in `test-integration`, so a
  one-line rule change costs a full infra startup to verify.
- Tests that depend on each other's leftover state, or on being run in a
  particular order, so a single target can't be re-run or narrowed.
