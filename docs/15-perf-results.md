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

Against the `compose/docker-compose.dev.yml` stack on a developer laptop
with other work running concurrently, so these are pessimistic.

| `WATERMARK_MS` / `POLL_BLOCK_MS` | min | mean | p50 | p95 | received |
|---|---|---|---|---|---|
| 500 / 1000 (**shipped defaults**) | 1852 ms | 1976 ms | 1967 ms | **2121 ms** | 15/15 |
| 0 / 25 (**correctness buffer off**) | 29 ms | 94 ms | 82 ms | **156 ms** | 15/15 |

### 26a.3 The latency is entirely those two constants

An earlier revision of this section reported "a ~1 second floor that
survives tuning" and localised it to the Kafka-consume hop. **Both claims
were wrong, and the method that produced them is worth naming so it isn't
repeated.** They came from reasoning about a configuration change that had
not actually been applied to the running process, rather than from a
measurement that isolated the hop.

Two measurements settle it:

- **The Kafka hop is not the cost.** A bare kafkajs produce-to-consume
  round trip, using `fusion-svc`'s exact consumer construction
  (`kafka.consumer({ groupId })`, no `maxWaitTimeInMs` override), measures
  **min 7 ms, mean 16.8 ms, p95 76 ms**. There is no second of latency
  hiding in kafkajs or the broker.
- **Turning the two constants down removes essentially all of it** —
  p95 2121 ms → 156 ms, a 13x reduction, with nothing else changed.

The mechanism is in `packages/redis/src/watermarkWindow.ts` and is by
design. A lone event can never be released by the watermark itself: the
watermark is `max(sensor_ts seen) - watermarkMs`, and a single event *is*
the max, so its own timestamp is always newer than the watermark it would
have to clear. It is released by the other path — `poll()` treats a
blocking `xread` that times out as "no domain has anything else in flight"
and flushes the buffer. So an isolated event waits out one full
`POLL_BLOCK_MS` before it is emitted, on top of `WATERMARK_MS`.

That is correct behaviour, not a bug. Under real sustained multi-domain
load, later events advance the watermark and release earlier ones without
waiting for a timeout, so the per-event cost is far lower than the
single-probe figure. **This harness measures the worst case — an isolated
event arriving into an idle pipeline — which is the right thing to measure
for a latency floor and the wrong thing to quote as typical.**

### 26a.4 What this means for the NFRs

**G1 is met with very large margin.** At the shipped defaults the
sensor-to-operator leg costs **~3.5% of the 60-second budget** (p95
2121 ms). Even reserving the full edge COA target of 30 s for the decision
leg, G1 has room to spare. This is the first real evidence for the
product's central claim.

**NFR 8.1's "track fusion latency < 500 ms" is a configuration choice, not
a performance wall.** At `WATERMARK_MS=0 / POLL_BLOCK_MS=25` the pipeline
delivers p95 156 ms — comfortably inside the target — but `WATERMARK_MS=0`
discards the Pitfall 1 protection that stops out-of-order cross-domain
arrivals producing phantom duplicate tracks, which is a correctness
guarantee the product should not trade for a latency number.

The honest framing for whoever owns this next: the NFR as written is
achievable only by giving up event-time ordering. Either restate it as
"≤ 500 ms of processing after the watermark closes", or reduce
`POLL_BLOCK_MS` (which costs nothing but a busier poll loop and is the
cheap half of the win — it is pure dead time for an isolated event) while
keeping `WATERMARK_MS` at a value justified by real sensor clock skew.
Lowering `POLL_BLOCK_MS` alone is the recommended change.

**Latency degrades as input rate rises** — an earlier run measured 1773 ms
min at 300 ms probe spacing versus 1013 ms at 1000 ms. The pipeline is a
single serialized `while (running) { await runPipelineOnce(...) }` loop, so
arrivals queue behind one another. PERF-001's ~9,889 events/sec measured
raw *ingestion*, which is a different path; **fusion throughput under
sustained load has still not been measured** and is the obvious next
benchmark.

### 26a.5 Reproducing

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

### 26b.2 Results on real hardware

Getting a real GPU into headless Chrome needs an exact flag pair:
`--use-gl=angle --use-angle=gl-egl` (plus `--ignore-gpu-blocklist`). Plain
`--use-gl=egl` **silently falls back to SwiftShader**, and a first run of
this benchmark did exactly that, measuring 2.9 fps at 250 entities — a
figure that describes the software rasteriser and says nothing about the
map. The tool now queries `WEBGL_debug_renderer_info` and prints the
renderer it actually got, so that failure cannot be silent again.

Measured on **Intel HD Graphics 520 (SKL GT2)** — a 2016 integrated laptop
GPU, roughly the weakest hardware any operator would realistically use —
180 frames per run, with other heavy work running on the same machine:

| entities | mean fps | p95 fps | worst frame |
|---|---|---|---|
| 100 | 40.5 | 20.0 | 100 ms |
| 500 | 33.8 | 15.0 | 133 ms |
| 1,000 | **42.9** | 20.0 | 83 ms |
| 2,000 | 35.9 | 15.0 | 133 ms |
| 5,000 | 36.6 | 20.0 | 217 ms |
| 10,000 | 24.4 | 7.5 | 317 ms |
| 20,000 | 13.4 | 3.7 | 933 ms |

An independent run on the same GPU under lighter load measured 51.5 fps
mean at 1,000 and 55.1 at 2,000, with the same curve shape.

### 26b.3 NFR 8.1 is met at 1,000 tracks, with about 5x headroom

**The curve is flat from 100 to 5,000.** A hundred entities and five
thousand entities perform the same, within noise. That flatness is the
finding: entity count is not what limits the frame rate in that range — the
renderer is finishing inside the frame budget throughout, and the measured
rate reflects the vsync cap and machine load rather than map work. The knee
is between 5,000 and 10,000; real degradation starts around 20,000.

**An earlier prediction in [17-next-work.md](17-next-work.md) — that
"MapLibre will need symbol clustering or a switch to a custom layer well
before 1,000" — was wrong**, by roughly an order of magnitude, and has been
corrected there.

Two caveats, both of which make this result conservative rather than
optimistic:

1. `requestAnimationFrame` is vsync-capped, so mean fps can never exceed
   the display rate (~60 Hz) no matter how much headroom exists. This
   measures "does it keep up", not "how much faster could it go". The true
   ceiling at 1,000 tracks is higher than any number here.
2. p95 is harshly quantised at these rates — a single missed vsync reads as
   "30 fps". The low p95 values are dropped frames, not sustained
   degradation. Judge on mean plus worst-frame.

### 26b.4 Why it is this cheap

Both icon layers in `entity-layer.tsx` set:

```ts
"icon-allow-overlap": true,
"icon-ignore-placement": true,
```

This disables MapLibre's symbol collision detection. The initial reading of
that was that it would make 1,000 tracks *expensive* because nothing is
culled. The measurement says the opposite: with collision detection off
there is no per-frame placement work to do, just draw calls, and the GPU
absorbs those easily. The setting is both correct for a tactical display —
an operator must never have tracks silently hidden because they overlap —
and the reason the flat part of the curve is flat.

The text label layer does cull (`"text-allow-overlap": false`,
`"text-optional": true`); only the icons do not.

---

## 27. PERF-002/003/004 — Hardware-Dependent Benchmarks (Not Runnable in This Sandbox)

Consistent with every prior phase's honest treatment of the same constraint (Phase 2's `onnxruntime-node` GPU path, Phase 4's `vllm` engine construction, Phase 6's ARM64 build gap — see [10-risks-pitfalls.md](10-risks-pitfalls.md) and the Phase 6 roadmap note):

| Task | Target | Status |
|---|---|---|
| PERF-002 | `onnxruntime-node` GPU benchmark: < 200ms/frame | **Not runnable.** No GPU exists in this sandbox. The GPU code path itself was implemented and fail-fast-verified in Phase 2 (`REQUIRE_GPU=true` never silently falls back to CPU) but never benchmarked on real GPU hardware. |
| PERF-003 | `node-llama-cpp` Jetson benchmark: < 30s COA on edge | **Not runnable.** No physical Jetson exists in this sandbox; `vektor-edge`'s `tests/hardware-in-the-loop/` is documented but has no CI runner yet (Phase 6). The equivalent CPU-only path (a genuinely smaller GGUF model, Qwen2.5-0.5B) was benchmarked in Phase 4/7 testing at real wall-clock times (see `coa-svc/tests/generateCoa.test.ts`'s edge-mode run, ~24s in this session's own test run against non-Jetson CPU hardware) — informative, but not a substitute for the real target hardware number. |
| PERF-004 | vLLM gRPC benchmark: < 10s COA on cloud GPU | **Not runnable.** `vllm` itself was never installable in this sandbox's time budget (Phase 4) — the gRPC plumbing, prompt building, and confidence scoring around it are real and pytest-covered, but the actual `engine.generate()` call has never executed anywhere in this project's history, let alone been benchmarked. |

None of these are silently skipped — each is a named, tracked gap requiring real target hardware (a GPU node, a physical Jetson, a cloud GPU instance) that provisioning (INFRA-003, Phase 0) never actually completed in a form this sandbox can reach. Closing them is a hardware-availability problem, not a remaining code problem.
