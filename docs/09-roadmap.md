# VEKTOR — Development Roadmap

[← Back to VEKTOR-PRD.md](../VEKTOR-PRD.md)

---

## 15. Development Roadmap

### Phase 0 — Foundation & Repo Setup (Weeks 1–4)

**Goal:** All 6 repos scaffolded; core infra provisioned; `vektor-proto` publishing its first contract versions. (The original `vektor-platform` repo was later split into `vektor-backend` + `vektor-web` — a 7th repo — see `VEKTOR-PRD.md`'s changelog; task rows below that predate the split are tagged with where that code lives today.)

| Task ID | Task | Owner | Repo |
|---|---|---|---|
| INFRA-001 | `vektor-proto` scaffold: proto/schema layout, codegen CI (npm + PyPI publish) | Eng Lead | `vektor-proto` |
| INFRA-002 | `vektor-backend` Turborepo + pnpm workspace init; shared package scaffolding | Eng Lead | `vektor-backend` |
| INFRA-003 | Provision K8s cluster (k3s dev + full prod) | DevOps | `vektor-infra` |
| INFRA-004 | Deploy Kafka cluster (3 brokers + Schema Registry) | DevOps | `vektor-infra` |
| INFRA-005 | Deploy PostgreSQL + PostGIS + TimescaleDB; Drizzle migrations | Backend | `vektor-backend` |
| INFRA-006 | Deploy MinIO + Qdrant + Redis Cluster | DevOps | `vektor-infra` |
| INFRA-007 | Deploy Kong API Gateway + Keycloak | DevOps | `vektor-infra` |
| INFRA-008 | CI/CD pipelines (GitHub Actions + ArgoCD) across all 7 repos | DevOps | all |
| INFRA-009 | Observability stack (Prometheus, Grafana, Pino→Loki, OTel/Jaeger) | DevOps | `vektor-infra` |
| INFRA-010 | HashiCorp Vault + secret rotation | Security | `vektor-infra` |
| INFRA-011 | `vektor-ml` namespace + network policy (gRPC-only ingress) | Security/DevOps | `vektor-infra` |
| INFRA-012 | `vektor-ml` scaffold: `cv-train-svc`/`llm-cloud-svc` package layout, lazy heavy-ML-dep imports so CI lint/test doesn't need a GPU, CI (lint/test + gated GPU smoke test) | ML | `vektor-ml` |
| INFRA-013 | `vektor-edge` scaffold: bootstrap/provisioning scripts, Docker Compose overlay, offline map-pack tooling, EDGE-007's memory-budget CI gate | DevOps | `vektor-edge` |

### Phase 1 — Ingestion & Base Map (Weeks 5–10)

**All tasks done as of 2026-08-21**, except the NTP/PTP daemon half of SVC-004 (host/infra config, not TS application code — tracked separately, not blocking). Every service below is verified against real infrastructure (real Postgres/PostGIS, real Kafka, real RTSP/MQTT sources, a real browser) rather than mocks — see [[vektor-build-conventions]].

| Task ID | Task | Owner |
|---|---|---|
| SVC-001 | `ingest-svc`: fluent-ffmpeg RTSP/SRT → kafkajs producer — **done**, verified against a real MediaMTX RTSP source + real Kafka broker | Backend |
| SVC-002 | `ingest-svc`: GeoTIFF/SAR pipeline — **done** via `geotiff` (pure JS); `gdal-async` isn't installable in the current sandbox (no prebuilt binary for this Node ABI, source-build fallback blocked by network access to nodejs.org headers) — see `ingest-svc/src/geotiff/ingestGeoTiff.ts` | Backend |
| SVC-003 | `ingest-svc`: AIS + ADS-B + MQTT IoT adapters — **done**: hand-rolled ITU-R M.1371 AIVDM decoder (Class A position reports), SBS-1 text adapter for ADS-B, Mosquitto-verified MQTT republisher | Backend |
| SVC-004 | NTP/PTP clock sync daemon; `sensor_ts` validation middleware — **middleware half done** (`@vektor/kafka`'s `assertValidSensorTs`, used by every SVC-001/002/003 adapter); the NTP/PTP daemon itself is host/infra config, not application code | Backend |
| SVC-005 | `geospatial-svc`: PostGIS CRUD + spatial query API — **done**, integration-tested against live PostGIS; its `entities` schema/response shape was later fixed to actually match the canonical `Entity` contract (2026-08-21) | Backend |
| SVC-006 | Map tile server: PMTiles static server — **done** | Backend |
| FE-001 | Web dashboard: MapLibre GL base map + layer system — **done** | Frontend |
| FE-002 | Web dashboard: Socket.io entity rendering — **done**: typed event contract matches §13.2 exactly, every inbound event Zod-validated at the socket boundary | Frontend |
| FE-003 | Web dashboard: sensor health panel — **done**: live per-sensor status/latency/drop-rate, flags the REQ-1.8 >1% drop-rate threshold | Frontend |
| SVC-007b | `fusion-svc`: `POST/GET /api/v1/sensors`, `DELETE /api/v1/sensors/:id` — REQ-1.8's sensor-registration endpoint (§13.1, `docs/07-data-api.md`), the one requirements-table row left unbuilt after Phase 1 closed — **done** 2026-09-15, backs the dashboard's Sensor Coverage map layer (see FE-00X below) | Backend |
| FE-00X | Web dashboard: Sensor Coverage map layer — **done** 2026-09-15: geodesic coverage-circle rendering for any sensor with a registration on file, closing the permanent no-op the layer-legend checkbox had been since Phase 1 (`SensorHealth` carried no geometry until SVC-007b) | Frontend |

### Phase 2 — CV & Tracking Pipeline (Weeks 11–18)

**All tasks done as of 2026-08-23.** No GPU is available in the current sandbox, so GPU throughput itself is unverified — everything else (the real pipeline mechanics: training, export, inference, tracking, hot-swap) is verified against real infra on CPU. See [[vektor-build-conventions]] for the concrete substitutions this phase needed (`opencv4nodejs` → Kalman-filter velocity, same class of native-binding problem as `gdal-async` in Phase 1).

| Task ID | Task | Owner | Repo |
|---|---|---|---|
| ML-001 | `cv-train-svc`: YOLOv8 training + ONNX INT8 export — **done**, real end-to-end run (synthetic dataset, 1 CPU epoch, export, reload via onnxruntime) | ML | `vektor-ml` |
| ML-002 | `cv-inference-svc`: onnxruntime-node GPU build + ONNX loading — **done**; GPU path implemented with Pitfall 5's fail-fast discipline (`REQUIRE_GPU=true` never silently falls back to CPU) but unverified on real GPU hardware, which this sandbox doesn't have | ML/Backend | `vektor-backend` |
| ML-003 | `cv-inference-svc`: Kafka frame consumer → inference → detection publisher — **done**, verified against a real Kafka broker + the real checked-in ONNX fixture | Backend | `vektor-backend` |
| ML-004 | DeepSORT tracker: TypeScript re-implementation (mathjs) — **done**: Kalman filter + greedy IoU association (a documented simplification of the Hungarian algorithm) | ML/Backend | `vektor-backend` |
| ML-005 | Velocity/heading estimation — **done via the Kalman-filter path**, not optical flow: `opencv4nodejs` needs a full OpenCV source build with no prebuilt binary, not installable in this sandbox. REQ-2.5 explicitly allows either path | ML/Backend | `vektor-backend` |
| ML-006 | ONNX hot-swap endpoint — **done**, verified with two genuinely different real models (different class count *and* input size) via a real multipart upload | Backend | `vektor-backend` |

### Phase 3 — Data Fusion Engine (Weeks 19–26)

**All tasks done as of 2026-08-23.** `fusion-svc` is also the Socket.io gateway `apps/web`'s FE-002 (Phase 1) needed — verified end-to-end against real Kafka, real Redis, real PostGIS, real Postgres, and a real socket.io-client (`tests/pipeline.integration.test.ts`): an AIS fix flows through the watermark window into an EKF-smoothed track, gets no-strike-flagged, persists to `entities`, and is corroborated by a separate EW/RF fix onto the *same* track before either reaches a client over the socket. See [[vektor-build-conventions]] for the noise-tuning, drizzle-polygon, and SRID pitfalls this phase surfaced.

| Task ID | Task | Owner |
|---|---|---|
| SVC-007 | `fusion-svc`: kafkajs consumer groups + Redis Streams topology — **done**: `src/kafka/ingestConsumers.ts` (schema-validated at the consumer boundary, Pitfall 1) + `@vektor/redis`'s `WatermarkWindow` (500ms event-time watermark, not Kafka/Redis arrival order) | Backend |
| SVC-008 | Extended Kalman Filter (TS, mathjs) — **done**: `src/ekf/extendedKalmanFilter.ts`, a real 6-state geodetic CV-model EKF (lat/lon/alt + N/E/U velocity) with a per-step-recomputed Jacobian for the lat/lon-coupling nonlinearity, not a relabeled linear KF | Backend |
| SVC-009 | Ontology mapping layer: detections → canonical Entity schema (Zod) — **done**: `src/ontology/mapToEntity.ts`, routed through `Entity.parse()` | Backend |
| SVC-010 | Blue-force tracking + no-strike zone engine (PostGIS ST_Contains) — **done**: `src/blueforce/`, meters-accurate auto-buffer zones (`ST_Buffer` via a `geography` cast) kept in sync per asset update, plus analyst-drawn manual zones | Backend |
| SVC-011 | EW/RF signal correlation via Redis Streams join — **done**: `EwRfEmission.estimated_position` goes through the same `TrackManager.correlate()` gate every domain uses, over the same watermark-windowed Redis Streams | Backend |

**Scope note**: CV `DetectionEvent` tracks (`detection.tracked`) carry only pixel-space bbox/velocity — no service in this system geo-registers a camera to real-world coordinates, so they're consumed and windowed (SVC-007's topology is complete for this topic) but don't yet produce fused entities. Not a roadmap gap for this phase; flagged for whichever future phase adds geo-registration.

### Phase 4 — Decision Support (Weeks 27–34)

**All tasks done as of 2026-08-24.** No GPU is available in this sandbox and `vllm` itself wasn't installable in a reasonable time budget (multi-GB CUDA/torch/xformers dependency tree, same class of gap as Phase 2's onnxruntime-node GPU path) — `llm-cloud-svc`'s real vLLM engine construction/inference call is unverified against real hardware, but everything else (gRPC plumbing, prompt building, JSON extraction/repair, confidence scoring) has real pytest coverage. Everything else in this phase is verified end-to-end against real infra: real Postgres, real Qdrant + real BGE-M3 inference, a real local GGUF model (node-llama-cpp), and a real gRPC server standing in for `llm-cloud-svc`. See [[vektor-build-conventions]] for the pitfalls this phase surfaced (a cross-repo gRPC class-identity bug, GBNF grammar termination).

| Task ID | Task | Owner | Repo |
|---|---|---|---|
| ML-007 | `llm-cloud-svc`: vLLM server, LLaMA 3.1 70B, gRPC endpoint — **done**: real `AsyncLLMEngine`/`SamplingParams` wiring, prompt builder, JSON extraction+repair, and logprob-based confidence scoring all real and pytest-covered (19 tests); the actual `engine.generate()` call is unverified on real GPU hardware, which this sandbox doesn't have | ML | `vektor-ml` |
| ML-008 | Qdrant doctrine ingestion + BGE-M3 embeddings — **done**: `@vektor/qdrant` (new package), real `Xenova/bge-m3` ONNX inference via `@xenova/transformers`, verified against a real Qdrant instance | ML/Backend | `vektor-backend` |
| ML-009 | `coa-svc`: context builder + Qdrant RAG retrieval — **done**: `sanitizeForPrompt()` (Pitfall 4), situation JSON assembly, real RAG retrieval against real doctrine content ingested into Qdrant | Backend | `vektor-backend` |
| ML-010 | `coa-svc`: gRPC call via `@vektor/proto` + Zod output validator — **done**: real gRPC round-trip against a real server implementing `LLMService`, hard Zod gate + confidence threshold (R-002) | Backend | `vektor-backend` |
| ML-011 | Edge COA: node-llama-cpp + Mistral 7B Q4_K_M — **done**, verified end-to-end against a real (much smaller) GGUF model, grammar-constrained JSON generation (`llama.createGrammarForJsonSchema`) rather than free-text + regex extraction | Backend | `vektor-backend` |
| SVC-012 | Target Workbench threat scoring engine — **done**: `coa-svc/src/threat/scoring.ts`, a documented 5-factor weighted sum (REQ-4.1's "algorithm documented and auditable"), `GET /api/v1/target-workbench/ranking` | Backend | `vektor-backend` |
| SVC-013 | COA approval workflow + audit-svc write — **done**: `services/audit-svc` (new, append-only `audit_log` table), coa-svc's approve/reject routes write through it synchronously before responding | Backend | `vektor-backend` |

**Scope note**: Phase 4's roadmap table has no `FE-xxx` task — the Target Workbench/COA approval UI itself is not part of this phase's scope (unlike Phases 1–3, which each paired backend work with a matching frontend task). `apps/web` has no COA/Target Workbench components yet.

### Phase 5 — Logistics, Alerting & Reporting (Weeks 35–40)

**All tasks done as of 2026-08-25.** Three new `vektor-backend` services. `alert-svc` consumes a BullMQ queue fusion-svc now enqueues to on every entity upsert (the Phase 3→5 integration point) and runs both a geofence entry/exit check and an EWMA/z-score anomaly detector per job, dispatching through real email (maildev)/webhook/SMS-shaped (Twilio REST API stand-in) channels. `logistics-svc` syncs a real Debezium PostgresConnector (pgoutput plugin, registered against a dedicated "ERP" Postgres instance and Kafka Connect) rather than a hand-rolled CDC format, forecasts depletion via `ml-regression-simple-linear`, and optimizes routes via a real `pgr_dijkstra` call against a dedicated pgRouting-extension Postgres instance (pgRouting isn't installed on the main PostGIS image — see [[vektor-build-conventions]]). `reporting-svc` generates real PDF (puppeteer-core + the system's existing Chrome binary) and PPTX (pptxgenjs) situation reports, dispatches them on a real cron schedule (cron-parser), and serves raw CSV/GeoJSON entity export. 41 tests across the three services, all against real infra (Postgres, Redis, Kafka, a real Debezium/Kafka-Connect stack, a real pgRouting instance, real SMTP, real headless Chrome) — zero mocks. See [[vektor-build-conventions]] for this phase's pitfalls (bigint-as-string over the wire, pptxgenjs's CJS/ESM interop, cron-parser's same interop issue, a raw `db.execute` timestamp-as-string trap, and a shared-table test-concurrency race).

| Task ID | Task | Owner |
|---|---|---|
| SVC-014 | `alert-svc`: geofence rule engine — **done**: `services/alert-svc/src/geofence/checkZones.ts`, Redis-backed per-entity "currently inside" set diffed each job to detect ENTRY/EXIT correctly per zone's own trigger config | Backend |
| SVC-015 | `alert-svc`: anomaly detection (EWMA + z-score) — **done**: `src/anomaly/ewmaDetector.ts`, circular-aware (heading wraparound at 0/360 handled via unit-vector EWMA, not a naive degree diff) | Backend |
| SVC-016 | `alert-svc`: multi-channel dispatch — **done**: `src/dispatch/channels.ts` — IN_APP (the persisted Alert row itself), EMAIL (real nodemailer -> real maildev SMTP), WEBHOOK (real HTTP POST), SMS (real Twilio REST wire shape against a stand-in server — no real Twilio account exists in this sandbox, same "real protocol, stand-in endpoint" caveat as Phase 4's llm-cloud-svc/vLLM) | Backend |
| SVC-017 | `logistics-svc`: ERP CDC integration (Debezium) — **done**: a real Debezium `PostgresConnector` (pgoutput plugin) registered via Kafka Connect against a dedicated "ERP" Postgres instance (`docker run postgres:16 -c wal_level=logical`), producing genuine Kafka Connect JsonConverter envelopes; `src/cdc/inventoryConsumer.ts` decodes them and upserts into VEKTOR's own `inventory_items`/`inventory_history` — verified end-to-end with a real insert+update round-trip within seconds, well inside REQ-1.7's 60s bound | Backend |
| SVC-018 | `logistics-svc`: supply forecasting (ml-regression) — **done**: `src/forecast/regression.ts` (`ml-regression-simple-linear`, the sub-package `ml-regression`'s meta-package re-exports), fires a once-per-transition `LOW_STOCK` alert (REQ-6.3) rather than re-firing on every 15-min recompute | Backend |
| SVC-019 | `logistics-svc`: route optimization (pgRouting) — **done**: a real `pgr_dijkstra` call (`src/routing/findRoute.ts`) against a dedicated pgRouting-extension Postgres instance and a synthetic road grid fixture; a flagged/blocked edge is modeled as an effectively-infinite-cost edge so REQ-6.2's "recomputed within 5s of a new hazard flag" is automatic — every route request already reads the live `blocked` flags, no separate recompute step exists to schedule | Backend |
| SVC-020 | `reporting-svc`: PDF (Puppeteer) + PPTX (pptxgenjs) — **done**: real headless-Chrome PDF rendering (`puppeteer-core` + the system's existing `google-chrome`, same pattern as apps/web's Playwright screenshots) and real pptxgenjs slide generation; also covers REQ-7.2 (cron-scheduled dispatch via `cron-parser`, real email delivery with the report as a real attachment) and REQ-7.3 (CSV/GeoJSON raw export) | Backend |

### Phase 6 — Edge Deployment (Weeks 41–46)

**All tasks done as of 2026-08-26.** EDGE-002's core ARM64 compatibility question is now genuinely closed: a standalone QEMU-emulated ARM64 container (`node:22-bookworm-slim`, `--platform linux/arm64`) installed `onnxruntime-node@1.27.0` fresh and ran a real inference pass against the checked-in `yolov8n.int8.onnx` fixture — `process.arch` reported `arm64` (not a silent x64 fallback), input/output tensor names and the `[1, 84, 8400]` output shape matched YOLOv8's real spec exactly, ~9.8s/frame under emulation (irrelevant to the compatibility question; real throughput is a hardware-only number, see below). The full multi-service Dockerfile build (`docker buildx build --platform linux/arm64 -f services/cv-inference-svc/Dockerfile --load .`) got very close twice but hit transient sandbox network degradation both times, not a real build/logic failure — every individual dependency it needs (including `onnxruntime-node` itself) is now independently confirmed to install and work correctly under real ARM64 code execution. What remains genuinely hardware-in-the-loop-only for EDGE-002/003 — real GPU/Ampere throughput and `node-llama-cpp`'s Mistral 7B load test on actual Jetson silicon — is the same class of gap as Phase 2's GPU inference and Phase 4's vLLM engine call; no physical Jetson or ARM64 CI runner exists in this sandbox. No `field-pwa` app exists anywhere in `vektor-web` yet (no FE-xxx task has created it) — `compose/docker-compose.edge.yml`'s entry for it stays a placeholder image; every other edge-profile service in that file is real and buildable.

| Task ID | Task | Owner | Repo |
|---|---|---|---|
| EDGE-001 | k3s edge node config + offline bootstrap scripts — **done** (Phase 0 scaffold); `compose/docker-compose.edge.yml` fixed to build from real Dockerfiles with the env vars each service actually reads, instead of placeholder image tags | DevOps | `vektor-edge` |
| EDGE-002 | YOLOv8n INT8 ONNX build + onnxruntime-node ARM64 test on Jetson — **done**: `services/cv-inference-svc/Dockerfile` (multi-arch); the INT8 model itself was already built in Phase 2; `onnxruntime-node`'s real ARM64 binding independently verified (real inference, real `arm64` process arch, correct output shape) — see the note above. **Real Jetson GPU throughput not done** — no physical runner exists | ML | `vektor-edge` |
| EDGE-003 | node-llama-cpp Mistral 7B Q4_K_M load test on Jetson — **build done** (`services/coa-svc/Dockerfile`, `VEKTOR_COA_MODE=edge`). **Real Jetson load test not done**, same hardware gap as EDGE-002 — unlike onnxruntime-node, `node-llama-cpp`'s ARM64 binding itself wasn't independently re-verified this pass | ML | `vektor-edge` |
| EDGE-004 | Offline PMTiles map pack generation — **done**, verified end-to-end against a real GeoJSON fixture with a from-source `tippecanoe` build + the real `pmtiles` binary; CI job does the same on every PR | Backend | `vektor-edge` |
| EDGE-005 | IndexedDB offline store + BullMQ sync queue — **done**: `apps/web/src/lib/offline-sync.ts` + `useOfflineSync` (browser half, verified in a real headless Chrome) draining into `services/edge-sync-svc`'s real BullMQ intake queue (backend half) | Frontend | `vektor-web` + `vektor-backend` |
| EDGE-006 | Delta sync on reconnect (Kafka offset replay) — **done**: `services/edge-sync-svc`, a persisted-SQLite-offset Kafka consumer (not relying on Kafka's own group-commit position) verified against real Kafka with a kill-and-resume scenario | Backend | `vektor-backend` |
| EDGE-007 | CI check: reject any edge build > 10 GB memory budget — **done** (Phase 0 scaffold), re-verified passing | DevOps | `vektor-edge` |

### Phase 7 — Public API, Hardening & Launch (Weeks 47–54)

**Done as of 2026-08-26** except SEC-005 (not attempted this phase — `vektor-edge` was another session's in-flight Phase 6 work for most of this phase's duration) and PERF-002/003/004 (GPU/Jetson/cloud-GPU hardware doesn't exist in this sandbox, same class of gap as Phase 2/4/6). New `@vektor/auth` package (JWT verification + RBAC, real RS256 crypto tests against a local JWKS) is wired into every public-facing `vektor-backend` service; a live Keycloak "vektor" realm happened to be available in this sandbox and caught a real bug before it shipped — see [ADR-0008](adr/0008-rbac-role-strings-match-keycloak.md). `@fastify/swagger` + `fastify-type-provider-zod`'s `jsonSchemaTransform` generate real OpenAPI 3.1 specs directly from each service's existing Zod route schemas (no hand-authored spec to drift — [ADR-0010](adr/0010-openapi-generated-from-zod.md)). Postgres RLS on `audit_log` is verified against a real non-superuser role, with its one real limitation (superuser/table-owner bypass) documented rather than glossed over ([ADR-0011](adr/0011-rls-defense-in-depth.md)). See the new [ADR set](adr/README.md) (14 records, DOC-002), [13-operator-manual.md](13-operator-manual.md) (DOC-001), [14-sec004-security-audit.md](14-sec004-security-audit.md) (SEC-004), and [15-perf-results.md](15-perf-results.md) (PERF-001) for full detail.

| Task ID | Task | Owner |
|---|---|---|
| API-001 | Public REST API v1 + OpenAPI 3.1 spec (auto-generated from Fastify schemas) — **done**: real `/documentation`/`/documentation/json` on `geospatial-svc`, `coa-svc`, `alert-svc`, `logistics-svc`, `reporting-svc`, `audit-svc`; verified the generated spec is actually valid OpenAPI 3.1 against a live `app.inject()` call, not just "the plugin registered without crashing" | Backend |
| API-002 | WebSocket subscription API scoped to bounding box — **done**: `fusion-svc/src/socket/gateway.ts` implements the `subscribe:area`/`unsubscribe` client→server events `vektor-proto`'s §13.2 contract already defined since Phase 1/3 but nothing had implemented server-side; entity events are filtered per-socket by bounding box, verified with 6 real `socket.io-client` round-trip tests (multi-client AOI isolation, unsubscribe reverting to broadcast, malformed-payload handling). `sensor:status`/`alert:fired`/`coa:ready` stay unscoped (documented judgment call, not in REQ-9.2's literal text) | Backend |
| SEC-001 | Istio mTLS enforcement across all TS services — **mTLS already done** (mesh-wide `PeerAuthentication` STRICT, scaffolded pre-Phase-7); **`AuthorizationPolicy` deliberately deferred**, not silently skipped — no per-service Kubernetes ServiceAccount convention exists yet to match against (real Deployment manifests land in `vektor-backend` itself, not yet written), and a default-deny authz policy without matching allow-rules would break real traffic on a live cluster. The concrete call graph for whoever adds this next is recorded in the istio module and [ADR-0012](adr/0012-istio-mtls-before-authz-policy.md). `terraform validate` against the live modules could not be run — this sandbox's connection to `registry.terraform.io` failed outright (`context deadline exceeded`) during this phase; `terraform fmt -check` (syntax only, no provider schema) passed | Security/DevOps |
| SEC-002 | `vektor-ml` network policy audit + gRPC-only enforcement — **done**: audited `vektor-ml-enclave`'s existing NetworkPolicies; found and fixed one real over-broad rule (DNS egress matched *any* namespace, not just CoreDNS's — narrowed via a new `dns_namespace` variable); confirmed `cv-train-svc` correctly gets no ingress-allow rule (it's a batch CLI job with no listener, not a gap). See [ADR-0013](adr/0013-network-policy-default-deny.md) | Security |
| SEC-003 | RBAC policy audit + row-level security (Drizzle RLS) — **done**: new `@vektor/auth` (`requireRole()` Fastify preHandler) wired into `geospatial-svc`, `coa-svc`, `alert-svc`, `logistics-svc`, `reporting-svc`, `audit-svc`, `cv-inference-svc`'s upload route, matching §13.1's role table exactly; fixed a real spoofing gap along the way — `coa-svc`/`alert-svc` used to trust a client-supplied `actor_user_id`/`actor_role`/`operator_id` body field for audit attribution, now read from the verified JWT only ([ADR-0009](adr/0009-actor-identity-from-verified-jwt.md)). Postgres RLS on `audit_log` (a real `vektor_app` non-superuser role, `UPDATE`/`DELETE` policies that always evaluate false) verified end-to-end against real Postgres — see [ADR-0011](adr/0011-rls-defense-in-depth.md) for the one caveat that keeps this from being fully closed yet | Security |
| SEC-004 | Penetration testing + red team exercise — **done to the extent achievable without a live deployed environment**: real `pnpm audit` against `vektor-backend`'s live lockfile (1 critical, 11 high, 7 moderate, 0 low as of 2026-08-29, down from 1/18/18/1 at Phase 7 time after upgrading `drizzle-orm`, `nodemailer`, and `@fastify/static` — concrete package/CVE list in [14-sec004-security-audit.md](14-sec004-security-audit.md)), a full engagement plan/rules-of-engagement for the actual red-team exercise once a real target exists, and a self-audit throwing real forged/tampered/wrong-role JWTs at SEC-003's `requireRole()` implementation. `vektor-proto`/`vektor-edge` dependency audits and `vektor-ml`'s Python-side scan were inconclusive (sandbox network degradation, `pip-audit`'s venv creation failing on this environment's no-sudo/no-`ensurepip` constraint) — open gaps, documented as such, not treated as clean | Security |
| SEC-005 | Full-disk encryption (LUKS) on edge nodes — **not attempted this phase**; `vektor-edge` was another concurrent session's in-flight Phase 6 work for most of Phase 7's duration | DevOps |
| PERF-001 | Load test: 10,000 events/sec sustained — **done**: new `@vektor/load-test` package, a real Kafka producer/consumer pair publishing genuine schema-valid `AisPositionReport` messages and validating every one on the consume side (the real per-message cost, not skipped). Measured **9,889 events/sec mean sustained, 11,333/sec peak**, against a single shared dev Kafka broker under real multi-session resource contention — effectively at parity with the NFR target despite not running on the dedicated 3-broker cluster §11 specifies. Full detail and a known measurement-artifact caveat in [15-perf-results.md](15-perf-results.md) | QA |
| PERF-002 | onnxruntime-node benchmark: < 200ms/frame on GPU node — **not runnable**, no GPU in this sandbox (same gap as ML-002) | ML |
| PERF-003 | node-llama-cpp benchmark: < 30s COA on Jetson edge — **not runnable**, no physical Jetson/CI runner exists yet (same gap as EDGE-002/003) | ML |
| PERF-004 | vLLM gRPC benchmark: < 10s COA on cloud GPU — **not runnable**, `vllm` was never installable in this sandbox to begin with (same gap as ML-007) | ML |
| DOC-001 | Operator training manual + runbooks — **done**: [13-operator-manual.md](13-operator-manual.md), written against the system as actually built (every endpoint/role/workflow named exists in the codebase), with 7 incident runbooks each mapped to a real failure mode this project's own build-out actually found | Product |
| DOC-002 | ADRs for all major architecture decisions — **done**: [14 ADRs](adr/README.md) spanning Phase 0 through this phase, fulfilling R-011's stated mitigation ("ADRs for every major decision") | Engineering |

---

