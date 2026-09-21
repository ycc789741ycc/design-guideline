# Decision Records

Code records *what* a system does. It almost never records *why* it does it
that way, which option was rejected, or what the team knew at the time. That
context decays fastest — it lives in a thread, a meeting, or one person's
head — and its absence is what makes teams re-litigate settled questions and
"clean up" a constraint that was load-bearing.

An Architecture Decision Record (ADR) is the fix: one short Markdown file per
decision, committed next to the code, written once and then frozen.

## What gets a record

Write one when the decision is **costly to reverse** or **non-obvious to the
next reader**:

- Choosing or replacing a datastore, broker, queue, or cache.
- A service boundary — what gets split out, what stays merged.
- An auth, tenancy, or data-isolation model.
- A public API or event contract that other teams build against.
- Adopting or dropping a framework, language, or major dependency.
- Deliberately accepting a trade-off — a known limit, a temporary
  workaround, a deviation from this guideline.

Don't write one for a choice that a later PR could simply undo, for
formatting or style (that's [naming conventions](naming-conventions.md) and
the linter), or for something the code already states plainly. A rule of
thumb: if a competent new joiner would read the code and ask *"why on earth
is it like this?"*, that question deserves an ADR.

## Where they live

```
docs/
└── decisions/
    ├── README.md     # index: one line per ADR, newest last
    ├── template.md   # copy this to start a new one
    ├── 0001-use-postgres-for-the-primary-store.md
    └── 0002-split-billing-into-its-own-service.md
```

`docs/decisions/` at the repository root, not `doc/adr/` or `doc/arch/`. It
reads as plain language rather than jargon, it sits under the same `docs/`
tree as everything else, and it is what current ADR tooling (MADR and
friends) expects — so an off-the-shelf tool works without configuration.

**Which repo.** A decision scoped to one service lives in that service's
repo, so it versions with the code that implements it. A decision binding
more than one repo lives in the guideline repo instead, and once it has
settled it also becomes a rule under `base/` — the rule states what to do,
the ADR keeps the reasoning that produced it.

## Naming and numbering

- `NNNN-short-title-in-kebab-case.md` — four digits, zero-padded, starting
  at `0001`.
- Take the next free number when you open the PR. If two land at once, the
  later one to merge renumbers; a number is never reused.
- Numbers are permanent identifiers. Never renumber a merged ADR, even when
  it is superseded — links and code comments point at it.
- The title is the decision, not the topic: `0007-drop-redis-session-store`,
  not `0007-sessions`.

## Status lifecycle

| Status | Meaning |
|---|---|
| `Proposed` | Open for discussion; the PR is the discussion. |
| `Accepted` | In force. Build as it says. |
| `Rejected` | Considered and declined. The file stays. |
| `Deprecated` | No longer relevant; nothing replaced it. |
| `Superseded by 0014` | Replaced. Always names the successor. |

**An accepted ADR is immutable.** Fix a typo, add a link, update the status
line — but never rewrite the context, decision, or consequences after it
merges. A record you can edit is just documentation, and documentation
edited in hindsight loses the one thing an ADR is for: what was actually
known and intended at the time.

Changed your mind? Write a new ADR that supersedes the old one, and edit the
old one's status to point forward. Both files stay. Rejected ADRs stay too —
knowing an option was weighed and why it lost prevents someone from
proposing it again next quarter.

## What goes in one

Six sections, in this order — copy
[`examples/decision-record-template.md`](examples/decision-record-template.md)
to start, and see
[`examples/decision-record-example.md`](examples/decision-record-example.md)
for a filled-in one:

1. **Title** — `# 0004. Use Postgres for the primary store`.
2. **Status** — one of the values above, plus the date it reached it.
3. **Context** — the forces in play: constraints, requirements, what you
   knew, what you didn't. No solution yet.
4. **Decision** — active voice, present tense, as a decision already made:
   *"We use Postgres…"*, not *"We should"* or *"We propose"*.
5. **Consequences** — what becomes easier **and what becomes harder**. An
   ADR with only upsides in this section hasn't been thought through; the
   costs you name here are what a future reader needs most.
6. **Alternatives considered** — each with the reason it lost. A bare list
   of names is worthless; *"MySQL — no native `jsonb`, and our event payloads
   are schemaless"* is the useful part.

Keep it to one page. If it needs more, the decision is probably several
decisions.

## Keeping them honest

- The ADR ships in the **same PR** as the change it justifies, not
  afterwards. A decision documented later is documented from memory.
- A reviewer can ask for an ADR as a review comment, the same as asking for
  a test. "This needs an ADR" is a legitimate blocking comment.
- Where code exists *because of* a decision and that isn't obvious locally,
  link it: `// see docs/decisions/0007-drop-redis-session-store.md`.
- Keep `docs/decisions/README.md` current in the same PR — an index nobody
  updates is how a decision log rots.
