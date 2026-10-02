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

Every branch is cut from exactly one of three places, decided by the kind of
change and whether it belongs to an epic:

| Change kind | Base to branch from |
|---|---|
| `feature`, `bugfix`, `docs`, `chore`, `refactor`, `test`, `epic` | The repository's mainline integration branch — `develop` if the repo has one, otherwise `master`/`main`. |
| `feature`, `bugfix`, `docs`, `chore`, `refactor`, `test` that is part of an epic | That epic's branch (e.g. `origin/epic/PROJ-1200/billing-rewrite`), **not** mainline — see [Epic branches](#epic-branches). |
| `hotfix` | The existing released version being fixed — that release's branch or tag (e.g. `release/1.4`, `v1.4.2`), **not** mainline. |

- Branch from the remote-tracking ref you just fetched (e.g.
  `origin/develop`), not from whatever your local checkout happens to
  point at.
- Name the branch as described in [Branch naming](#branch-naming) below,
  and create it as a worktree — see
  [Cutting the branch](#cutting-the-branch-use-a-worktree).
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
| `epic` | Integrating a multi-phase change built on several sub-branches (see [Epic branches](#epic-branches)). Never holds direct work. |
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
- An `epic` branch carries the epic's own tracker key; each of its
  sub-branches carries its own child ticket's key, not the epic's.

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
epic/PROJ-1200/billing-rewrite
```

Counter-examples, and why they fail:

```
feature/user-export              # no ticket segment
PROJ-1234/user-export            # no change kind
feature/1234/user-export         # tracker key stripped of its prefix
feature/PROJ-1234/UserExport     # not kebab-case
feature/PROJ-1234/fix/export     # slash inside the description
```

## Cutting the branch: use a worktree

When a new branch is cut from the current one, create it as a **git
worktree** rather than switching branches in the shared clone:

```
git fetch origin
git worktree add ../<repo>-PROJ-1234 -b feature/PROJ-1234/user-export origin/develop
```

A clone has one working tree, and `git switch` moves it for everyone and
everything pointed at that directory. When more than one worker — two
people, or (increasingly) several coding agents running concurrently — is
active in the same checkout, that single tree is shared mutable state:

- One switching branches pulls the files out from under another mid-edit,
  mid-build, or mid-test-run, and the failure looks like a code bug rather
  than a checkout race.
- Uncommitted work from one task gets committed onto the other's branch,
  or stashed and lost.
- Build output, caches, and test databases in the tree are rebuilt against
  whichever branch won the race.

A worktree gives each task its own directory with its own checked-out
branch, sharing one object store and one set of refs. Commits, fetches,
and branches are visible across all of them; the files are not.

### Rules

- One worktree per branch or task. Git refuses to check the same branch
  out in two worktrees — lean on that, don't work around it with
  `--force`.
- Create it from the freshly fetched remote ref, the same as any other
  branch (see [Where a branch comes from](#where-a-branch-comes-from)),
  and name the branch by the normal convention.
- Put worktrees in a sibling directory outside the repository
  (`../<repo>-<ticket>`), not nested inside the working tree, so they
  can't be picked up by builds, test discovery, linters, or an accidental
  `git add`. If a nested location is unavoidable, git-ignore it.
- Git-ignored local setup does **not** come along: `.env`, installed
  dependencies, virtualenvs, and local data all have to be re-created in
  the new worktree per the repo's normal setup steps. Copy `.env` from
  your existing checkout or regenerate it from `.env.example` — it stays
  local and git-ignored in the new worktree too, exactly as it is in the
  old one.
- Always remove the worktree as soon as you finish modifying its branch —
  once the work is committed and pushed — not when the branch is later
  merged: `git worktree remove <path>`, then `git worktree prune`. See
  [Removing the worktree](#removing-the-worktree-when-the-work-is-done).
- The one case for switching in place is a checkout you are certain is
  yours alone at that moment. Whenever concurrent work is possible —
  and it always is when an agent is running in the repo — cut a worktree
  instead.

### Removing the worktree when the work is done

A worktree exists for one stretch of modification on one branch. When
that stretch ends, remove it — every time, without waiting for review or
merge:

```
git -C ../<repo>-PROJ-1234 status        # nothing uncommitted
git -C ../<repo>-PROJ-1234 push          # nothing unpushed
git worktree remove ../<repo>-PROJ-1234
git worktree prune
```

- "Done" means the modifications are committed and pushed to `origin`
  (or the branch is abandoned). Check with `git status` and that the
  branch has no commits ahead of its upstream before removing.
- Never pass `--force` to get past a refusal. `git worktree remove`
  refuses when there are uncommitted or untracked changes — commit and
  push them, or deliberately discard them, then remove.
- Remove the worktree, not the branch. The branch stays on `origin` (and
  locally) for review and merge; deleting the branch is a separate step
  after merge.
- If more changes are needed later — review feedback, a failing CI run —
  fetch, then cut a fresh worktree from the pushed branch
  (`git worktree add ../<repo>-PROJ-1234 feature/PROJ-1234/user-export`),
  pull, re-create its git-ignored setup, and remove it again when that
  round of changes is pushed.
- An agent that created a worktree removes it before it reports the task
  finished; leaving one behind is an unfinished task.

Worktrees left around until merge pile up: each holds its own copy of
`.env` and installed dependencies, goes stale against `origin`, and keeps
its branch checked out so nobody else can check it out elsewhere. Removing
them on completion keeps the set of live worktrees equal to the set of
work actually in progress.

## Epic branches

Some work is planned in several stages or phases, each made of features
that are orthogonal to one another — separable enough to be built,
reviewed, and tested on different branches, but not meant to reach
mainline piecemeal. Give that work an **epic branch**: one integration
branch for the whole plan, with each orthogonal piece on its own
sub-branch beneath it.

```
develop (mainline)
└── epic/PROJ-1200/billing-rewrite
    ├── feature/PROJ-1201/invoice-model
    ├── feature/PROJ-1202/invoice-api
    └── test/PROJ-1203/invoice-e2e
```

A single branch for the whole plan would grow into one unreviewable change;
merging each piece to mainline as it lands would ship half a plan. The epic
branch keeps each review small while mainline only ever sees the finished
whole.

### When to use one

- The plan has multiple stages or phases, and the features inside them are
  orthogonal enough to develop on separate branches in parallel or in
  sequence.
- The pieces shouldn't reach mainline one at a time — partially landed,
  they'd leave mainline inconsistent or expose an unfinished capability.
- A change that fits one ticket and one branch never gets an epic. If each
  piece is independently shippable, skip the epic and branch each one off
  mainline as usual.

### Flow

1. **Cut the epic from mainline.** Fetch, then create the epic branch on
   `origin` straight from the fetched mainline ref. Nothing is modified,
   so no worktree is needed:

   ```
   git fetch origin
   git push origin origin/develop:refs/heads/epic/PROJ-1200/billing-rewrite
   ```

2. **Cut each sub-branch from the epic.** One sub-branch per orthogonal
   feature or phase, named by the normal convention with its own child
   ticket, cut as a worktree from the fetched epic ref:

   ```
   git fetch origin
   git worktree add ../<repo>-PROJ-1201 -b feature/PROJ-1201/invoice-model origin/epic/PROJ-1200/billing-rewrite
   ```

   All the worktree rules above apply unchanged — including removing it as
   soon as the sub-branch's work is pushed.

3. **Merge each sub-branch back into the epic** by PR targeting the epic
   branch, not mainline. The same review, test, and CI gates apply as for
   a PR into mainline. A later phase that depends on an earlier one is cut
   from the epic after the earlier phase has merged into it.

4. **Merge the epic into mainline** once every sub-branch has landed, as a
   single PR from the epic branch. It must pass both test tiers like any
   other PR. Delete the epic branch after the merge.

### Epic branch rules

- No direct commits on an epic branch. Every change arrives through a
  sub-branch PR; the only other commits are merges from mainline.
- Keep the epic current: merge mainline into it regularly, and always
  before opening the final PR, so conflicts surface early and in small
  pieces. Never rebase or force-push an epic branch — other sub-branches
  are built on top of it.
- Sub-branches follow [Before modifying a branch](#before-modifying-a-branch)
  against the epic: fetch and pull the epic before modifying, and pull its
  latest changes into the sub-branch when another sub-branch has landed
  and you need them.
- A defect in epic work found before the epic merges is fixed on a
  `bugfix` sub-branch off the epic, not on mainline — the code it fixes
  doesn't exist on mainline yet.
- Hotfixes are unaffected: they still branch from the release and merge
  back into mainline, and reach open epics on their next merge from
  mainline.

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
