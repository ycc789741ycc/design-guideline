# Principles

Core values that inform every other document in this guideline. When a
specific rule doesn't cover a situation, fall back to these.

## Values

- **Explicit over clever.** Code should be understandable by the next person
  (or AI tool) without needing to hold the whole system in their head.
- **Consistency over local optimization.** A slightly worse pattern applied
  consistently beats a better pattern applied inconsistently, because
  consistency is what makes a codebase predictable to both humans and AI.
- **Readability over micro-performance**, unless a profiler has shown
  otherwise. Don't guess at performance problems.
- **Boring technology by default.** Reach for a new dependency, pattern, or
  paradigm only when the boring option has a demonstrated, specific
  shortcoming — not because it's newer.
- **Small, reversible changes over big, risky ones.** Prefer shipping in
  increments that are easy to review and easy to roll back.

## Decided trade-offs

Document trade-offs the overrides has already made, so they don't get
re-litigated in every PR:

- _Example: "We accept slightly higher latency on read paths in exchange for
  simpler caching logic."_
- _Example: "We prioritize type safety over development speed for anything
  touching billing or auth."_

Replace the examples above with your overrides' actual decided
trade-offs as they come up.
