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
  `git switch -c feature/user-export origin/develop`), not from whatever
  your local checkout happens to point at.
- Name the branch `<change-kind>/<short-description>`; for a hotfix include
  the version being patched (`hotfix/1.4.2-token-refresh`).
- A repo has exactly one mainline. If both `develop` and `master`/`main`
  exist, `develop` is the mainline and `master`/`main` is release history —
  don't branch day-to-day work off release history.

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
