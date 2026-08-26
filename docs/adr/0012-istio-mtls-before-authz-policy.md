# ADR-0012: Ship Istio STRICT mTLS now; defer `AuthorizationPolicy` until per-service identities exist

**Status:** Accepted (Phase 7 / SEC-001, 2026-08-26)

## Context

SEC-001 calls for "Istio mTLS enforcement across all TS services." A mesh-wide `PeerAuthentication` with `mtls.mode: STRICT` was already scaffolded in `terraform/modules/istio` — this closes the *authentication* half of §14.1's "Unauthorized data access" mitigation (every sidecar-to-sidecar connection must be mutually authenticated; plaintext is rejected outright, not merely upgraded opportunistically). The *authorization* half — which specific caller identity may call which specific service — needs Istio `AuthorizationPolicy` resources matched against each workload's Kubernetes ServiceAccount.

## Decision

Ship STRICT mTLS now. Do not add `AuthorizationPolicy` resources in this phase. No ServiceAccount-per-service naming convention exists anywhere in this repo yet — actual Kubernetes Deployment manifests (and therefore ServiceAccounts) are expected to land in `vektor-backend`'s own repo per-service, following the same "this repo is bootstrap/ops tooling, application manifests land next to the application" split `vektor-edge`'s README already documents for its own placeholder image tags.

## Consequences

- Writing `AuthorizationPolicy` rules against guessed ServiceAccount names now would be unverifiable configuration that might not even match what eventually gets deployed — worse than no policy, since it would read as "done" in a terraform diff without being a real control.
- A default-deny `AuthorizationPolicy` specifically must never be added without its matching allow-rules landing in the *same* change: applied alone to a real cluster, it would immediately block every request into `vektor-backend`, including Kong's own ingress traffic.
- The concrete call graph this phase's SEC-003/API-001 work established by reading actual source (not guessed) is recorded in the istio module's own comments for whoever adds this next: `coa-svc → audit-svc` (REST), `coa-svc → fusion-svc` (REST), `vektor-gateway (Kong) → every public-facing service` (REST ingress). `fusion-svc → alert-svc` is BullMQ-over-Redis, not HTTP, and has no Istio-mesh `AuthorizationPolicy` analog.
- Until `AuthorizationPolicy` lands, the security posture inside the mesh is "any authenticated (mTLS) sidecar may call any other" — identity is verified, but not yet scoped. This is the accepted interim state, not a false sense of completeness: SEC-001's roadmap entry covers mTLS specifically, and this ADR is the record of why authorization scoping is a named follow-up rather than silently skipped.
