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
