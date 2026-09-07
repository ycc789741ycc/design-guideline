# API Design (example-org override — extend)

This **extends** [`base/backend/api-design.md`](../../../base/backend/api-design.md) —
all base rules still apply. This file adds example-org-specific rules on
top; it does not restate the base content.

## Addition: internal services use gRPC, not REST

The base guideline assumes REST for its examples. For example-org:

- All internal (service-to-service) APIs are defined as gRPC services with
  `.proto` files checked into a shared `protos/` repository — external-
  facing APIs (public API, webhooks) remain REST per base rules.
- `.proto` files follow `package example_org.<service>.v1;` naming.
- Breaking changes to a `.proto` message require a new `v2` package, same
  versioning philosophy as base's URL-path versioning for REST.
- Every gRPC service implements the standard gRPC health-checking protocol
  in addition to the base guideline's HTTP health/readiness endpoints.
