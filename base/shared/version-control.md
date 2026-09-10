# Version Control

## Before modifying a branch

- Always fetch and pull the latest upstream (`origin`) changes before making
  any modifications to a branch — whether continuing work on an existing
  branch or cutting a new one from it. Never commit on top of a stale base.
- If the pull surfaces conflicts or diverging history, resolve that first;
  don't paper over it with a force-push.

This keeps local work from silently diverging from what's already landed
upstream, and avoids conflicts and duplicated work that only surface late,
at review or merge time.

## Where a branch comes from

Every branch is cut from exactly one of two places, decided by the kind of
change:

| Change kind | Base to branch from |
|---|---|
| `feature`, `bugfix`, `docs`, `chore`, `refactor`, `test` | The repository's mainline integration branch — `develop` if the repo has one, otherwise `master`/`main`. |
| `hotfix` | The existing released version being fixed — that release's branch or tag (e.g. `release/1.4`, `v1.4.2`), **not** mainline. |

- Branch from the remote-tracking ref you just fetched (e.g.
  `git switch -c feature/PROJ-1234/user-export origin/develop`), not from
  whatever your local checkout happens to point at.
- Name the branch as described in [Branch naming](#branch-naming) below.
- A repo has exactly one mainline. If both `develop` and `master`/`main`
  exist, `develop` is the mainline and `master`/`main` is release history —
  don't branch day-to-day work off release history.

## Branch naming

Every branch is named:

```
<change-kind>/<ticket>/<short-description>
```

Three segments, always in that order, always lowercase except the ticket
key. The shape is fixed so that tooling — MR templates, changelog
generation, tracker integrations, release notes — can parse a branch name
without guessing.

| Change kind | Use for |
|---|---|
| `feature` | New behavior or a user-visible capability. |
| `bugfix` | Correcting broken behavior on mainline. |
| `hotfix` | Fixing a released version out-of-band (see [Hotfixes](#hotfixes)). |
| `refactor` | Restructuring without changing behavior. |
| `docs` | Documentation only. |
| `chore` | Build, tooling, dependency bumps, housekeeping. |
| `test` | Test-only additions or repairs. |

Pick the kind by what the change *does*, not by which team asked for it.
If a branch would honestly need two kinds, it's doing two things — split
it.

### The ticket segment

- Use the tracker key exactly as the tracker renders it (`PROJ-1234`),
  including its case. Don't abbreviate it, don't strip the project prefix,
  and don't invent a number.
- Work starts from a ticket. If there isn't one, open it first — that's
  where the requirement, the reviewer context, and the audit trail live.
- For the rare change with genuinely no ticket, use `no-ticket` in that
  slot (`chore/no-ticket/pin-ci-image`) so the name still parses. Treat
  it as an exception worth explaining in the MR, not a default.
- One ticket may have several branches; a branch belongs to exactly one
  ticket.

### The description segment

- Lowercase kebab-case, roughly two to four words: `user-export`,
  `token-refresh-retry`, `drop-legacy-webhook`.
- Describe the change, not the ticket title verbatim and not the files
  touched.
- No slashes inside it — the third segment is the last one, so a stray
  slash breaks the parse.
- For a `hotfix`, lead the description with the version being patched:
  `hotfix/PROJ-1188/1.4.2-token-refresh`.

### Examples

```
feature/PROJ-1234/user-export
bugfix/PROJ-1290/duplicate-invoice-email
hotfix/PROJ-1188/1.4.2-token-refresh
refactor/PROJ-1301/split-billing-service
docs/PROJ-1312/version-control-flow
chore/PROJ-1320/bump-postgres-driver
test/PROJ-1333/checkout-integration-coverage
```

Counter-examples, and why they fail:

```
feature/user-export              # no ticket segment
PROJ-1234/user-export            # no change kind
feature/1234/user-export         # tracker key stripped of its prefix
feature/PROJ-1234/UserExport     # not kebab-case
feature/PROJ-1234/fix/export     # slash inside the description
```

## Hotfixes

- A hotfix exists because a released version is broken in an environment
  that cannot wait for the next mainline release. It branches from that
  released version so the fix ships without dragging along unreleased
  mainline work.
- Keep a hotfix minimal and scoped to the defect — no refactors, no
  drive-by cleanups, no dependency bumps. Anything beyond the fix belongs
  on a normal branch off mainline.
- A hotfix is not done when it ships. Merge it back into mainline (and into
  any newer release line still supported) so the next release doesn't
  regress the fix. If the merge back is deferred, it is tracked as work,
  not left to memory.
- The same review, test, and CI gates apply to a hotfix as to any other
  change. Urgency is a reason to keep the change small, not a reason to
  skip verification.
