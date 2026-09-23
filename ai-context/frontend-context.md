# Frontend Context (condensed from base/frontend/)

Read `shared-context.md` first — applies here too.

- **Build/test in containers**: the bundle build, unit tests, `lint`,
  `typecheck` and `scan` run inside a container via `make` targets with
  the Node/tooling version pinned by the image — no host `npm`/`pnpm`
  invocation in docs or CI. The hot-reload dev server runs as
  `make start-app MODE=dev` (source bind-mounted via `compose.dev.yaml`);
  running it natively is the one permitted exception, only when the
  containerized one genuinely can't work: it keeps its standard target
  name, states in a comment why, and pins the host Node version.
  Either way the dev server's host port is the repo's uncommon block
  (e.g. `24811`), never the framework default (`3000`, `4200`, `5173`)
  — see Local ports in `shared-context.md`.
- **Component hierarchy**: atomic (no business logic, pure props) →
  composite (local UI state only) → feature (owns data fetching/business
  logic) → page (wires features together). Dependencies flow downward
  only — atomic never imports from features.
- **State management**: local UI state → component state. Server/remote
  data → a dedicated data-fetching layer with caching (not plain global
  state). Global client state → only for things genuinely needed
  everywhere (theme, current user, flags). Don't store derived state —
  compute it.
- **Accessibility (baseline, always)**: full keyboard operability, color
  never the only signal, WCAG AA contrast, real `<label>` elements (not
  placeholder-only), native semantic elements over `<div onClick>`, focus
  management for modals.
- **Interaction patterns**: every async action has a visible loading
  state. Errors shown near the point of failure with actionable copy.
  Every list has a designed empty state. Destructive actions require
  confirmation naming the specific item affected.
- **Configuration**: client-exposed values come from `.env` via the
  framework's public prefix (`VITE_`, `NEXT_PUBLIC_`), never a literal in
  source. Anything with a public prefix is public — no API keys or tokens
  there; proxy them through a backend route. Prefer run-time config over
  build-time inlining for anything that differs per environment.

## Example pattern (see base/frontend/examples/good-component.md for full code)

Generic atomic `Button` (no domain knowledge) → generic composite
`ConfirmDialog` (reusable across features) → feature component that owns
the actual API call and business logic (e.g. `useCancelOrder`).
