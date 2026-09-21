# Decision Record Template

Copy this file to `docs/decisions/template.md` in your repo, then copy *that*
to `docs/decisions/NNNN-short-title-in-kebab-case.md` for each new decision.
Delete the `<!-- -->` guidance comments as you fill it in.

---

```markdown
# NNNN. <The decision, as a short statement>

<!-- Title is the decision itself, not the topic:
     "Use Postgres for the primary store", not "Database choice". -->

- **Status:** Proposed | Accepted | Rejected | Deprecated | Superseded by NNNN
- **Date:** YYYY-MM-DD
- **Deciders:** <roles or teams, not just individuals>

## Context

<!-- The forces in play at the time: requirements, constraints, deadlines,
     team size, what you knew and what you didn't. Written so someone who
     joins in two years understands the situation without asking anyone.
     No solution here yet — just the problem and its pressures. -->

## Decision

<!-- Active voice, present tense, already decided:
     "We use ...", not "We should use ..." or "We propose ...".
     Say what is in force, precisely enough to build from. -->

We ...

## Consequences

**Easier**

- <!-- What this unlocks or simplifies. -->

**Harder**

- <!-- What this costs: new failure modes, added operational burden,
     doors it closes. A record with nothing in this list is unfinished. -->

**Accepted risks**

- <!-- Known problems you are choosing to live with, and the trigger that
     would make you revisit. Omit the section if there are none. -->

## Alternatives considered

- **<Option>** — <why it lost. The reason is the point; a bare name is not
  an alternative considered.>
- **<Option>** — <...>
```
