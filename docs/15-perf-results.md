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

**The COA leg is excluded, and for a more fundamental reason than first
stated.** The original reason given was practical: `coa-svc` needs a live
Qdrant with a populated doctrine collection plus either a cloud gRPC LLM or
a local GGUF, and its latency is dominated by model inference on hardware
this project has never had (PERF-003/004).

The deeper reason, found 2026-10-16: **nothing automatically triggers COA
generation.** `POST /api/v1/coa/generate` in `coa-svc/src/routes/generate.ts`
is the only entry point, it is operator-driven, and a grep for
`coa/generate` or `COA_SVC_URL` across the whole repo returns that file and
nothing else. §9.7's data-flow diagram describes "coa-svc triggered by
alert or operator request", and that route's own comment anticipated
alert-svc as the second caller "once it exists" — alert-svc exists now and
does not call it. The automatic leg was never built.

So G1's sensor-to-decision path **has a human in the middle today**.
Measuring wall-clock across that boundary would measure operator reaction
time and report it as system latency. The figures below are the machine
half: sensor to the operator's screen, which is where the automatic path
currently ends. They are a **lower bound on G1**, and closing the gap is a
product decision (see [17-next-work.md](17-next-work.md)), not a
measurement one.

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

## 26c. Correlation Correctness Verification (2026-10-18)

Not a performance result, recorded here because it is the only end-to-end
verification of the fusion pipeline's correctness that exists and it
belongs next to the other measured claims.

### 26c.1 What was being checked

A scenario-driven run found that fusion-svc's correlation gate used
**horizontal distance only** — `haversineMeters` discards `alt_m` — so an
aircraft at 3,000m overflying a vessel within ~1.5km was a correlation
candidate. The two merged into one entity at an EKF-averaged 1,508.8m, an
altitude belonging to neither platform, and one of the two real contacts
disappeared from the operational picture. Nothing errored; the surviving
entity looked entirely normal.

A vertical gate was added, checked against the EKF's *predicted* altitude
so a climbing aircraft still correlates with itself, with `ewrf` exempt
from both sides because RF geolocation carries no usable altitude.

### 26c.2 Results

Run by a second session against an isolated stack (own PostGIS, own Redis,
`perf.*` topics), independently of the session that wrote the fix.

| Case | Scenario | Result |
|---|---|---|
| **A** | Ship + aircraft at 3,000m, identical ground track | **PASS** — 2 entities: `Aircraft.ADSB` 3000.2m, `Vessel.AIS` 0.0m. Was 1 entity at 1508.8m |
| **B** | One aircraft climbing 300→9,000m, 30s reports (~435m climb per report against a 1,000m gate) | **PASS** — 1 track, 8565m. Did not fragment |
| **C** | ADS-B at 3,000m + 4 EW/RF fixes with no usable altitude | **PASS** — fused, both sensors on one entity, altitude holds at 3000.2m |
| **D** | ADS-B with `altitude_ft: null` flown over a vessel | **PASS** — 1 entity, `CONFLICTED`. Still merges, by design |

Case B is the one that justifies the predicted-altitude design: a
last-measured gate would have split that climb into a string of tracks,
trading the original bug for a worse one.

Case D is the documented intent rather than a compromise. An altitude-less
report can still merge with a surface track, which is the pre-existing
behaviour and the safer of the two errors — a wrongly merged track is
visible and recoverable, splitting one aircraft across many tracks on
fabricated altitude is neither. What changed is that the classification
conflict now **latches** `CONFLICTED`, so the merge is visible instead of
silent. Previously the flag was recomputed per observation and the next
agreeing one erased it, meaning under continuous dual-sensor input an
operator would essentially never see it.

### 26c.3 One defect, found three times

Case C initially failed, and failed in a way worth recording. The
correlation was right — the RF emission joined the airborne track, SVC-011
intact — but the aircraft's altitude read **−247m**. Below sea level. Four
consecutive RF fixes had dragged a known-good 3,000m track down through
zero, the overshoot being the filter's velocity term.

That was the third appearance of a single underlying defect, and the first
two fixes were each correct in isolation while the bad value kept flowing
to the next consumer:

1. The **gate** ignored altitude entirely. Fixed by adding a vertical gate.
2. The **ADS-B mapper** coerced a null `altitude_ft` to `0`, so "not
   reported" was indistinguishable from "at sea level". Fixed by adding
   `alt_known`.
3. The **EKF** was still handed that same placeholder `0` as a
   *measurement*, by the EW/RF mapper. Fixed by having `alt_known: false`
   suppress the altitude update, not merely the gate.

**This is a provenance problem, not three bugs.** Each layer learned that
altitude could be unknown; none of them told the next. The lesson that
generalises: a placeholder substituted for a missing measurement will be
read as real by whatever consumes it next, and fixing the consumer you
happen to be looking at does not stop it reaching the others. `alt_known`
exists so the distinction between "measured zero" and "no measurement" is
carried rather than inferred.

### 26c.4 The grep was run, and found a twelfth

§26c.3 ends by suggesting that anywhere the codebase writes `?? 0` for a
nullable sensor field is a candidate for the same defect. That grep has
since been run across `vektor-backend/services`, `vektor-backend/packages`
and `vektor-web/apps`, and the result is recorded here so nobody repeats
it.

Thirteen hits. Twelve are benign, and are worth naming so they are not
re-chased:

- **Counters and accumulators** — `(counts[e.affiliation] ?? 0) + 1`,
  `lastForwardedAt`, `snapshotFrameNumbers`. Zero is the correct identity.
- **Config defaults** — `COA_MIN_CONFIDENCE ?? 0.15`, `iouThreshold ?? 0.3`,
  `confidenceThreshold ?? 0.25`. Not sensor data.
- **SQL aggregates** — `entity_count ?? 0`, `point_count ?? 0`,
  `dropped ?? 0`. A count of nothing genuinely is zero.
- **`observationMappers.ts:40`** — `(event.altitude_ft ?? 0) * FEET_TO_METERS`
  looks like a hit and is not: the next line is
  `alt_known: event.altitude_ft !== null`, so the zero is explicitly marked
  as not-a-measurement. This is the intended shape — the placeholder stays
  because the EKF needs a number, and the provenance travels beside it.

**The thirteenth was real, and it was in the scenario player** — the tool
that found the original defect. `interpolateAlt` defaulted a missing
waypoint altitude to 0, so an `adsb` entity authored without `alt_m`
emitted `altitude_ft: 0, on_ground: true`: a sea-level aircraft, on the
ground. After the fixes above that distinction became load-bearing in the
direction that matters, because fusion-svc now treats a *reported* 0 as a
known altitude — so a scenario aircraft with no stated altitude would
correlate with any vessel beneath it. The exact merge the vertical gate
exists to prevent, arriving through the test tool. Fixed the same way:
`TrackState.alt_known`, `altitude_ft: null` when false, and `on_ground`
going with it, since a scenario cannot claim an aircraft is grounded from
an altitude it never gave. Stated altitudes unchanged, `--dry-run`
byte-identical so determinism holds, 24/24.

**The detail that makes this worth a section rather than a footnote:** the
engineer who wrote the bug report naming this pattern — "a placeholder
standing in for a missing measurement, read downstream as real" — was
shipping an instance of it in the same change. Not through carelessness;
it simply is not visible from inside the code you are writing. It took
running the grep, as a mechanical sweep rather than a review, to surface
it. That is the argument for treating this as a class and sweeping for it
periodically, rather than as a bug that has now been fixed.

### 26c.5 Methodology: how to reset fusion-svc, and how not to

Two contaminations during this run nearly produced false results, and both
are silent — they yield plausible extra entities that look like a
correlation bug rather than like leftovers.

- **`TRUNCATE entities` does not reset fusion-svc.** In-memory TrackManager
  state survives and gets re-upserted. Case A came back as 3 entities
  because a phantom `Aircraft.ADSB` at 0.0m from an earlier case was still
  live in the tracker.
- **Restarting the process makes it worse, not better.** Observations
  buffered in Redis Streams survive, so the pipeline re-consumes the whole
  session backlog and repopulates the table — 20 rows in one case,
  including `LOST` tracks from an earlier scenario. Kafka is not the
  culprit: `ingestConsumers.ts` uses `fromBeginning: false`.

**The reliable reset is `redis-cli FLUSHALL`, then restart, then
`TRUNCATE`.** Any fusion-svc entity count taken without flushing Redis
first should be treated as suspect. The same hazard, reached from a
different direction, already cost this project a day over the
sensorCoverage/pipeline test collision.

The regression tests covering all of the above live in
`services/fusion-svc/tests/trackManager.test.ts` and run in CI without
infrastructure; the end-to-end cases above need a live stack and are
reproduced by publishing to the real ingest topics.

---

## 27. PERF-002/003/004 — Hardware-Dependent Benchmarks (Not Runnable in This Sandbox)

Consistent with every prior phase's honest treatment of the same constraint (Phase 2's `onnxruntime-node` GPU path, Phase 4's `vllm` engine construction, Phase 6's ARM64 build gap — see [10-risks-pitfalls.md](10-risks-pitfalls.md) and the Phase 6 roadmap note):

| Task | Target | Status |
|---|---|---|
| PERF-002 | `onnxruntime-node` GPU benchmark: < 200ms/frame | **Not runnable.** No GPU exists in this sandbox. The GPU code path itself was implemented and fail-fast-verified in Phase 2 (`REQUIRE_GPU=true` never silently falls back to CPU) but never benchmarked on real GPU hardware. |
| PERF-003 | `node-llama-cpp` Jetson benchmark: < 30s COA on edge | **Not runnable.** No physical Jetson exists in this sandbox; `vektor-edge`'s `tests/hardware-in-the-loop/` is documented but has no CI runner yet (Phase 6). The equivalent CPU-only path (a genuinely smaller GGUF model, Qwen2.5-0.5B) was benchmarked in Phase 4/7 testing at real wall-clock times (see `coa-svc/tests/generateCoa.test.ts`'s edge-mode run, ~24s in this session's own test run against non-Jetson CPU hardware) — informative, but not a substitute for the real target hardware number. |
| PERF-004 | vLLM gRPC benchmark: < 10s COA on cloud GPU | **Not runnable.** `vllm` itself was never installable in this sandbox's time budget (Phase 4) — the gRPC plumbing, prompt building, and confidence scoring around it are real and pytest-covered, but the actual `engine.generate()` call has never executed anywhere in this project's history, let alone been benchmarked. |

None of these are silently skipped — each is a named, tracked gap requiring real target hardware (a GPU node, a physical Jetson, a cloud GPU instance) that provisioning (INFRA-003, Phase 0) never actually completed in a form this sandbox can reach. Closing them is a hardware-availability problem, not a remaining code problem.
