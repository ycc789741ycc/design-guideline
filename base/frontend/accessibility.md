# Accessibility

## Baseline (applies to every component)

- All interactive elements are reachable and operable via keyboard alone.
- Color is never the only signal (e.g. errors are marked with an icon/text,
  not red color alone).
- Text contrast meets WCAG AA (4.5:1 for normal text, 3:1 for large text)
  at minimum.
- Images have meaningful `alt` text; decorative images have empty `alt=""`.
- Form inputs have associated, visible labels — placeholder text is not a
  substitute for a label.

## Semantic markup

- Use native HTML elements (`<button>`, `<nav>`, `<label>`) over
  `<div onClick>` — native elements carry accessibility behavior for free.
- Heading levels (`h1`-`h6`) are used in order and reflect actual document
  structure, not chosen for visual size.

## Interaction

- Focus is managed explicitly for modals/dialogs (trap focus while open,
  return focus on close).
- Loading and error states are announced to screen readers (e.g. via
  `aria-live` regions), not just shown visually.

## Verification

- New composite/feature components are checked with an automated a11y
  linter/scanner as part of CI, plus a manual keyboard-only pass before
  merging anything with new interactive elements.
