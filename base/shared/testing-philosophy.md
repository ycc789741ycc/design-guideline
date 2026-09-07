# Testing Philosophy

## What "done" means

A change is not done until it has:
- Tests covering the new behavior (unit at minimum; integration where the
  change crosses a boundary — DB, network, another service).
- Existing tests still passing, with no tests deleted or weakened just to
  make CI green.
- Tests that assert on *behavior*, not implementation detail (avoid
  brittle tests that break on harmless refactors).

## Coverage expectations by risk tier

| Tier | Examples | Expectation |
|---|---|---|
| Core / stable | auth, payments, data models | High coverage, integration tests required, strict review |
| Standard features | most application logic | Unit tests required, integration tests where boundaries are crossed |
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
