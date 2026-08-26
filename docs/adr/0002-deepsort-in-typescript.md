# ADR-0002: Re-implement DeepSORT in TypeScript rather than call out to Python

**Status:** Accepted (Phase 2, 2026-08-23)

## Context

DeepSORT (Kalman filter + Hungarian-algorithm association) is the standard reference implementation for multi-object tracking, and every mature implementation is Python. `cv-inference-svc` is a TS service; §9.3/§9.9 restrict Python strictly to the two gRPC-only `vektor-ml` services (`cv-train-svc`, `llm-cloud-svc`) to keep the TS/Python boundary at exactly one crossing (`vektor-proto`'s `llm.proto`).

## Decision

Re-implement the tracker in TypeScript using `mathjs` for the Kalman filter, with a documented simplification: greedy IoU association instead of the Hungarian algorithm (optimal bipartite matching). This runs in-process in `cv-inference-svc`, immediately after ONNX inference, with no network hop.

## Consequences

- No second TS↔Python gRPC crossing for a per-frame hot path — avoids both the latency of an RPC per frame and a second class of cross-repo contract to maintain (Pitfall 3).
- Greedy IoU association can produce a suboptimal (not globally minimal-cost) assignment in dense, ambiguous-overlap scenes where the Hungarian algorithm would find a better global match. R-008 tracks this explicitly: "DeepSORT TypeScript re-implementation diverges from reference... mitigation: unit test against reference Python outputs; parity test suite in CI." No such parity suite exists yet as of Phase 7 — it's a real, open gap, not a closed one.
- `mathjs`'s `diag()` mirrors its input type (plain array in, plain array out — no `.toArray()`); wrap with `matrix(...)` first. A recurring interop trap for this specific library across both the CV tracker and the fusion EKF (ADR-0003).
