# Interaction Patterns

## Loading states

- Every async action (fetch, submit, navigation) has a visible loading
  state — never a silent, indefinite wait.
- Distinguish first-load (skeleton/spinner) from background refresh
  (subtle indicator, don't block the existing content).

## Error feedback

- Errors are shown near the point of failure (inline on a form field) not
  only as a generic global toast, unless the error is genuinely global
  (network down).
- Error copy explains what happened and, where possible, what to do next —
  not just an error code.

## Empty states

- Every list/collection view has a designed empty state — not a blank
  screen — explaining what would appear there and, where relevant, a call
  to action.

## Navigation

- Back/forward browser behavior is preserved (don't break history with
  client-side routing shortcuts).
- Destructive actions (delete, cancel subscription) require confirmation;
  the confirmation copy names the specific thing being affected, not a
  generic "Are you sure?".
