# Architecture Decision Records

[← Back to VEKTOR-PRD.md](../../VEKTOR-PRD.md) · [← Back to docs map](../README.md)

DOC-002 (Phase 7, roadmap §15) and R-011 (risk register §16: "Key engineers depart mid-project... Mitigation: ADRs for every major decision") — this directory is that mitigation.

These are not the forward-looking architecture *plan* (that's [04-system-design.md](../04-system-design.md)). Each ADR here records a decision actually made while building a phase, usually because reality (a missing native binding, a library's silent bug, a sandbox constraint, a security-audit finding) forced a choice the original PRD didn't anticipate. [10-risks-pitfalls.md](../10-risks-pitfalls.md) documents the *symptom and mitigation* for the subset of these that were live incidents; an ADR records the *decision and why*, for engineers who need to know whether a past choice still holds before changing it.

| # | Title | Phase | Status |
|---|---|---|---|
| [0001](0001-sensor-ts-ordering.md) | `sensor_ts`, never `kafka_received_at`, as the fusion ordering key | 1/3 | Accepted |
| [0002](0002-deepsort-in-typescript.md) | Re-implement DeepSORT in TypeScript rather than call out to Python | 2 | Accepted |
| [0003](0003-ekf-per-dimension-noise.md) | Per-dimension process/measurement noise in the geodetic EKF | 3 | Accepted |
| [0004](0004-native-binding-substitutions.md) | Substitute pure-JS/algorithmic alternatives when a native binding has no prebuilt binary | 1/2 | Accepted |
| [0005](0005-cross-repo-grpc-wrapper.md) | Runtime values crossing the `vektor-proto` `link:` boundary go through a wrapper module | 4 | Accepted |
| [0006](0006-debezium-cdc-not-hand-rolled.md) | Real Debezium CDC, not a hand-rolled change-detection poller | 5 | Accepted |
| [0007](0007-dedicated-pgrouting-instance.md) | pgRouting on a dedicated Postgres instance, not the main PostGIS database | 5 | Accepted |
| [0008](0008-rbac-role-strings-match-keycloak.md) | RBAC role strings match Keycloak's literal role names, spaces included | 7 | Accepted |
| [0009](0009-actor-identity-from-verified-jwt.md) | Actor identity for audit/approval actions comes from the verified JWT, never a request body field | 7 | Accepted |
| [0010](0010-openapi-generated-from-zod.md) | OpenAPI 3.1 is generated from existing Fastify/Zod schemas, not hand-authored | 7 | Accepted |
| [0011](0011-rls-defense-in-depth.md) | Postgres RLS as a DB-layer backstop for the audit log, on top of app-layer RBAC | 7 | Accepted |
| [0012](0012-istio-mtls-before-authz-policy.md) | Ship Istio STRICT mTLS now; defer `AuthorizationPolicy` until per-service identities exist | 7 | Accepted |
| [0013](0013-network-policy-default-deny.md) | Default-deny NetworkPolicy for the Python ML enclave, narrow named exceptions only | 0/7 | Accepted |
| [0014](0014-no-schema-duplication.md) | Every wire contract is defined exactly once, in `vektor-proto` | 0 | Accepted |
