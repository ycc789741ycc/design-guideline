# Decision Record — Worked Example

A filled-in ADR, for calibration on length and specificity. This is roughly
the upper bound: one page, concrete, and honest about costs. It would live at
`docs/decisions/0007-drop-redis-session-store.md`.

---

```markdown
# 0007. Drop the Redis session store in favour of signed cookies

- **Status:** Accepted
- **Date:** 2026-03-11
- **Deciders:** Platform team, with Security review

## Context

Sessions are held in Redis, keyed by an opaque cookie. Redis is in the
request path for every authenticated call, so it is a hard dependency for
availability: the two user-facing outages last quarter were both Redis
failovers, and neither involved a Redis feature we actually use.

Our sessions carry a user id, a tenant id, and a role — under 200 bytes,
written at login, read on every request, and otherwise never mutated. We do
not currently support forced logout of an active session, and no product
requirement has asked for it in three years.

We are about to add a second region, which would mean either cross-region
Redis replication or a second cluster with sticky routing.

## Decision

We store session state in a signed, encrypted cookie (AES-GCM, key from
`SESSION_KEY`) and remove Redis from the authentication path. Sessions
expire after 12 hours with no server-side record. Redis stays for rate
limiting and cache, where losing the data is harmless.

## Consequences

**Easier**

- Redis is no longer a hard dependency for authenticated traffic; a failover
  degrades rate limiting instead of logging everyone out.
- The second region needs no session replication and no sticky routing.
- One fewer stateful dependency in local dev and integration tests.

**Harder**

- Key rotation becomes a real operational procedure: we must accept the
  previous key for a 12-hour overlap, which means carrying two keys in
  config and a documented rotation runbook.
- Session payload is now size-constrained; anything beyond the current three
  fields needs a deliberate decision rather than "just add a key".

**Accepted risks**

- We cannot revoke an individual session before it expires. Mitigation: a
  global key roll invalidates everything, and the 12-hour lifetime bounds
  exposure. If a product requirement for per-session revocation lands, this
  decision must be revisited — a small revocation list would be the likely
  successor.

## Alternatives considered

- **Keep Redis, add cross-region replication** — solves nothing about the
  failover outages, and roughly doubles the operational surface we were
  trying to reduce.
- **Move sessions to Postgres** — removes the Redis dependency but puts a
  write-light, read-heavy table in the request path of every call, trading
  one availability coupling for another on a store we care about more.
- **Stateless JWTs with a refresh endpoint** — equivalent to this decision
  on the availability question, but adds a token-refresh flow and a second
  credential type for no benefit at our session lifetime.
```
