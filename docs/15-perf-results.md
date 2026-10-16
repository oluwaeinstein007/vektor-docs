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

## 26a. NW-013 — Sensor-to-Operator Latency (2026-10-16)

G1 ("sensor-to-decision latency < 60 seconds") is the product's headline
claim and nothing had ever measured it. Every component had been timed in
isolation; the composed path had not.

### 26a.1 Tool and scope

`vektor-backend/packages/latency-harness`. A real `AisPositionReport` is
produced to `dev.vektor.ais.position`; a **live `fusion-svc`** consumes it
through the full production path (Redis Streams event-time watermark
window, 6-state EKF, cross-sensor correlation, ontology mapping,
no-strike-zone check, Postgres upsert); a real `socket.io-client`
authenticated with a real Keycloak token receives the fused `Entity`. No
stub anywhere. Each probe carries a unique `sensor_id` which `fusion-svc`
carries through to the Entity's `source_sensors`, so a received entity maps
back to exactly the message that produced it rather than being matched by
timestamp.

**The COA leg is deliberately excluded.** `coa-svc` needs a live Qdrant with
a populated doctrine collection plus either a cloud gRPC LLM or a local
GGUF, and its latency is dominated by model inference on hardware this
project has never had (PERF-003/004). The figures below are therefore a
**lower bound on G1**.

### 26a.2 Results

25 samples per run, against the `compose/docker-compose.dev.yml` stack on a
developer laptop with eleven service images building concurrently in the
background — so these are pessimistic, not best-case.

| Configuration | min | mean | p50 | p95 | received |
|---|---|---|---|---|---|
| Defaults (`WATERMARK_MS=500`, `POLL_BLOCK_MS=1000`) | 1488 ms | 1914 ms | 1934 ms | **1965 ms** | 25/25 |
| Tuned (`WATERMARK_MS=100`, `POLL_BLOCK_MS=50`) | 1013 ms | 1085 ms | 1108 ms | **1155 ms** | 25/25 |

Probe-rate sensitivity, tuned configuration, 10 samples each:

| Probe interval | min | p95 |
|---|---|---|
| 300 ms | 1773 ms | 1915 ms |
| 1000 ms | 1013 ms | 1155 ms |
| 3000 ms | 1132 ms | 1349 ms |

### 26a.3 What this means

**G1 is comfortably met on this leg.** At p95 the sensor-to-operator path
consumes **1.9–3.3% of the 60-second budget**. Even allowing the edge COA
target of 30 seconds (NFR 8.1) for the decision leg, G1 has very large
headroom. This is the first real evidence for the product's central claim.

**NFR 8.1's "track fusion latency < 500ms" is NOT met**, and the reason is
worth stating precisely because most of it is policy, not slowness:

- ~1450 ms of the default-configuration figure is two deliberate constants:
  `WATERMARK_MS` (500) and `POLL_BLOCK_MS` (1000). The watermark exists to
  satisfy Pitfall 1 — never order by arrival time — and trading it away
  trades away correctness under sensor clock skew. Tuning both down removed
  almost exactly that much.
- A **~1 second floor remains after tuning**, and it is real rather than a
  measurement artifact: it does not shrink when probes are spaced three
  seconds apart. It is not the watermark (controlled for) and not the
  pipeline poll loop (`POLL_BLOCK_MS=50`), which localises it to the
  Kafka-consume-to-Redis-stream hop. `fusion-svc` constructs its consumer as
  `kafka.consumer({ groupId: "fusion-svc" })` with no `maxWaitTimeInMs`
  override, so kafkajs's defaults apply. **Confirming that is the obvious
  next step and was not done here.**
- **Latency degrades as input rate rises** — 1773 ms min at 300 ms spacing
  versus 1013 ms at 1000 ms. The pipeline is a single serialized
  `while (running) { await runPipelineOnce(...) }` loop, so arrivals queue
  behind one another. PERF-001 measured raw *ingestion* throughput at ~9,889
  events/sec, which is a different thing from the fused-entity path measured
  here; nothing has yet measured fusion throughput under sustained load.

Three follow-ups fall out of this, none of them blocking: identify the
remaining ~1s consumer-side floor; decide whether NFR 8.1's 500 ms target is
achievable at all with a correctness-preserving watermark, or whether the
NFR should be restated as "500 ms of processing after the watermark closes";
and measure the fusion path under sustained rate rather than single probes.

### 26a.4 Reproducing

```sh
pnpm stack:up
# start fusion-svc with the stack's env (see compose/.env.example)
KEYCLOAK_TOKEN=$(./scripts/dev-stack.sh token Analyst) \
  pnpm --filter @vektor/latency-harness run
```

---

## 26b. NW-014 — Map Render Performance (2026-10-16)

NFR 8.1 commits to "≥ 60fps on WebGL map with 1,000+ active tracks". The
dashboard had only ever been demonstrated with a handful of real entities.

### 26b.1 Tool

`vektor-web/apps/web/scripts/map-perf-bench.mjs`. Drives a real MapLibre GL
map in a real headless Chrome (`playwright-core` against the system binary —
the same no-bundled-browser pattern `reporting-svc` uses for
`puppeteer-core`), with the **same three symbol layers and identical
layout/paint expressions** as `src/components/entity-layer.tsx`, fed
features in the exact shape `src/lib/entity-geojson.ts` emits. It sweeps
entity counts, drives a continuous pan/zoom so every frame is a real
re-render, and reports mean/p95 frame times.

It is a layer-level benchmark, not a full-dashboard one — no Next.js, no
Socket.io, no HUD panels. That is deliberate: NFR 8.1 asks whether MapLibre
can render N entity symbols at 60fps with this project's symbology, and
mixing React re-render cost in would answer a different question.

### 26b.2 This environment cannot answer the question

No GPU is exposed to headless Chrome here, so WebGL runs on SwiftShader, a
software rasteriser. The measured result at only **250 entities** was
**2.9 fps mean, 1.6 fps p95, worst frame 1367 ms** — far below 60fps at a
quarter of the target load, on a renderer that is one to two orders of
magnitude slower than real hardware.

**That number says nothing about NFR 8.1.** It measures SwiftShader, not
the symbology. NW-014 therefore joins PERF-002/003/004 as
hardware-blocked: the tool is written, verified to drive the real code
path, and ready to produce a meaningful figure the moment it runs on a
machine with a GPU. Running it is a five-minute job for whoever has one.

### 26b.3 One finding that does not need a GPU

Both icon layers in `entity-layer.tsx` set:

```ts
"icon-allow-overlap": true,
"icon-ignore-placement": true,
```

This disables MapLibre's symbol collision detection, so **every symbol is
rendered with no culling** — there is no mechanism by which 1,000 tracks
cost less than 1,000 draws, however they are distributed on screen. It is
the right choice for a tactical display (an operator must not have tracks
silently hidden because they overlap), and it is also the single thing most
likely to decide whether the 1,000-track target is met. Whoever runs this on
real hardware should measure with and without it, and if the target is
missed, the options are symbol clustering at low zoom, a custom WebGL layer,
or restating the NFR.

The text label layer already sets `"text-allow-overlap": false` with
`"text-optional": true`, so labels do cull — only the icons do not.

---

## 27. PERF-002/003/004 — Hardware-Dependent Benchmarks (Not Runnable in This Sandbox)

Consistent with every prior phase's honest treatment of the same constraint (Phase 2's `onnxruntime-node` GPU path, Phase 4's `vllm` engine construction, Phase 6's ARM64 build gap — see [10-risks-pitfalls.md](10-risks-pitfalls.md) and the Phase 6 roadmap note):

| Task | Target | Status |
|---|---|---|
| PERF-002 | `onnxruntime-node` GPU benchmark: < 200ms/frame | **Not runnable.** No GPU exists in this sandbox. The GPU code path itself was implemented and fail-fast-verified in Phase 2 (`REQUIRE_GPU=true` never silently falls back to CPU) but never benchmarked on real GPU hardware. |
| PERF-003 | `node-llama-cpp` Jetson benchmark: < 30s COA on edge | **Not runnable.** No physical Jetson exists in this sandbox; `vektor-edge`'s `tests/hardware-in-the-loop/` is documented but has no CI runner yet (Phase 6). The equivalent CPU-only path (a genuinely smaller GGUF model, Qwen2.5-0.5B) was benchmarked in Phase 4/7 testing at real wall-clock times (see `coa-svc/tests/generateCoa.test.ts`'s edge-mode run, ~24s in this session's own test run against non-Jetson CPU hardware) — informative, but not a substitute for the real target hardware number. |
| PERF-004 | vLLM gRPC benchmark: < 10s COA on cloud GPU | **Not runnable.** `vllm` itself was never installable in this sandbox's time budget (Phase 4) — the gRPC plumbing, prompt building, and confidence scoring around it are real and pytest-covered, but the actual `engine.generate()` call has never executed anywhere in this project's history, let alone been benchmarked. |

None of these are silently skipped — each is a named, tracked gap requiring real target hardware (a GPU node, a physical Jetson, a cloud GPU instance) that provisioning (INFRA-003, Phase 0) never actually completed in a form this sandbox can reach. Closing them is a hardware-availability problem, not a remaining code problem.
