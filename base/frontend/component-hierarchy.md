# Component Hierarchy

## Levels

- **Atomic**: smallest reusable units with no business logic — buttons,
  inputs, icons. Purely presentational; receive everything via props.
- **Composite**: combinations of atomic components that form a recognizable
  unit — a form, a card, a modal. May hold local UI state (open/closed,
  focus) but not business/domain state.
- **Feature**: a self-contained slice of the product — checkout flow, user
  settings page. Owns its own data fetching and business logic, composed
  from composite and atomic components.
- **Page/route**: wires features together for a given URL; minimal logic of
  its own.

## Rules

- Atomic components never import from `features/` — dependencies flow
  downward only (page → feature → composite → atomic), never sideways or up.
- Shared/library components (used by more than one feature) live in a
  common `components/` directory; feature-specific components stay inside
  that feature's folder. See [`../../overrides/README.md`](../../overrides/README.md)
  for how reusability maps to review strictness.
- Props are the only way data flows into atomic and composite components —
  no reaching into global state from a supposedly-reusable component.

## Naming

- Component files match the component name in PascalCase:
  `UserAvatar.tsx` exports `UserAvatar`.
- Co-locate a component's styles, tests, and stories with the component
  itself rather than in parallel mirrored directories.
