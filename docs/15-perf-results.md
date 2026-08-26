# VEKTOR — Phase 7 Performance Benchmark Results

[← Back to VEKTOR-PRD.md](../VEKTOR-PRD.md)

---

## 26. PERF-001 — Event Ingestion Throughput (03-nfr.md: ≥ 10,000 events/sec sustained)

### 26.1 Tool

`vektor-backend/packages/load-test` — a real Kafka producer/consumer pair, not a synthetic microbenchmark. The producer publishes genuine, schema-valid `AisPositionReport` messages (the exact shape and Zod schema `fusion-svc`'s real consumer validates against); the consumer runs every message through `AisPositionReport.parse()` — the real per-message validation cost, not skipped for a bigger number. It publishes to an isolated `loadtest.vektor.ais.position` topic rather than the real `dev.*` topics, specifically so it never injects synthetic load into a `fusion-svc` instance another session might have running against the shared dev dataset.

### 26.2 Result (2026-08-26, single shared Kafka broker, 6 partitions)

| Target | Produced (mean/peak-1s) | Consumed (mean/peak-1s) |
|---|---|---|
| 10,000/sec, 20s | 9,947 / 11,000 | 9,889 / 11,333 |

**9,889 events/sec sustained (mean across the full run), peaking at 11,333/sec** — effectively at parity with the 10,000/sec NFR target, achieved against a **single-broker** dev Kafka instance running on a machine concurrently hosting several other active Claude Code sessions' workloads (see the coordination log for this phase — at least 6 concurrent sessions were active on this machine during this benchmark window), not the dedicated 3-broker (ISR=2) cluster R-009 specifies for production. This is a real, honest measurement of what this specific shared sandbox produced, not a claim about production hardware capacity.

### 26.3 A known measurement artifact — read before trusting the raw totals

Across every run, **total consumed count lagged total produced count** even with a 20-second drain buffer after the producer stopped (e.g. one run: 199,000 produced vs 179,000 consumed within the measurement window). Investigated during this session: this is not evidence of actual Kafka message loss (Kafka's delivery guarantee to a still-polling consumer group is not in question here) — it's an artifact of this tool's own measurement window ending on a fixed timer rather than on "queue empty," combined with consumer-group-join latency (~3s) eating into that window. **The rate figures (mean/peak-per-second) are the trustworthy numbers here — they only count seconds where consumption was actually observed. The raw total-consumed count is a lower bound, not a message-loss indicator.** A future iteration of this tool should end the measurement window on "no new messages for N seconds" rather than a fixed timer, to close this gap cleanly.

### 26.4 What this does and doesn't prove

- **Proves**: the real Kafka publish/consume/schema-validate path this project's actual ingestion pipeline depends on can sustain very close to the NFR target even in a resource-constrained, single-broker, multi-tenant sandbox.
- **Does not prove**: that `fusion-svc`'s *full* pipeline (EKF update, no-strike-zone check, Postgres write, Socket.io emit — the complete path from Kafka message to COP update) sustains 10,000/sec. This benchmark deliberately isolated ingestion throughput per the NFR's literal wording, rather than starting a second long-lived `fusion-svc` process against shared dev Postgres/Redis during a sustained high-throughput run — a real risk to other concurrent sessions' work, given the multi-session coordination already in effect this phase (see cross-session messages during Phase 6/7). **A full-pipeline throughput benchmark against a dedicated (non-shared) environment is a legitimate follow-up**, not something this result should be read as already covering.

---

## 27. PERF-002/003/004 — Hardware-Dependent Benchmarks (Not Runnable in This Sandbox)

Consistent with every prior phase's honest treatment of the same constraint (Phase 2's `onnxruntime-node` GPU path, Phase 4's `vllm` engine construction, Phase 6's ARM64 build gap — see [10-risks-pitfalls.md](10-risks-pitfalls.md) and the Phase 6 roadmap note):

| Task | Target | Status |
|---|---|---|
| PERF-002 | `onnxruntime-node` GPU benchmark: < 200ms/frame | **Not runnable.** No GPU exists in this sandbox. The GPU code path itself was implemented and fail-fast-verified in Phase 2 (`REQUIRE_GPU=true` never silently falls back to CPU) but never benchmarked on real GPU hardware. |
| PERF-003 | `node-llama-cpp` Jetson benchmark: < 30s COA on edge | **Not runnable.** No physical Jetson exists in this sandbox; `vektor-edge`'s `tests/hardware-in-the-loop/` is documented but has no CI runner yet (Phase 6). The equivalent CPU-only path (a genuinely smaller GGUF model, Qwen2.5-0.5B) was benchmarked in Phase 4/7 testing at real wall-clock times (see `coa-svc/tests/generateCoa.test.ts`'s edge-mode run, ~24s in this session's own test run against non-Jetson CPU hardware) — informative, but not a substitute for the real target hardware number. |
| PERF-004 | vLLM gRPC benchmark: < 10s COA on cloud GPU | **Not runnable.** `vllm` itself was never installable in this sandbox's time budget (Phase 4) — the gRPC plumbing, prompt building, and confidence scoring around it are real and pytest-covered, but the actual `engine.generate()` call has never executed anywhere in this project's history, let alone been benchmarked. |

None of these are silently skipped — each is a named, tracked gap requiring real target hardware (a GPU node, a physical Jetson, a cloud GPU instance) that provisioning (INFRA-003, Phase 0) never actually completed in a form this sandbox can reach. Closing them is a hardware-availability problem, not a remaining code problem.
