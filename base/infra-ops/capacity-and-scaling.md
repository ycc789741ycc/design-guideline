# Capacity & Scaling

## Principles

- Scaling decisions are driven by measured load, not guesswork — have the
  metrics before adding capacity or complexity.
- Prefer horizontal scaling of stateless components; treat statefulness as
  a deliberate, documented exception.

## Practices

- Load/capacity is reviewed ahead of known high-traffic events (launches,
  sales, marketing pushes) rather than reactively.
- Auto-scaling policies are tested (not just configured) — verify they
  actually trigger and recover as expected.
- Cost and capacity are reviewed together; scaling for headroom is
  intentional, not indefinite over-provisioning.
