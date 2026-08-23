# VEKTOR — Development Roadmap

[← Back to VEKTOR-PRD.md](../VEKTOR-PRD.md)

---

## 15. Development Roadmap

### Phase 0 — Foundation & Repo Setup (Weeks 1–4)

**Goal:** All 6 repos scaffolded; core infra provisioned; `vektor-proto` publishing its first contract versions.

| Task ID | Task | Owner | Repo |
|---|---|---|---|
| INFRA-001 | `vektor-proto` scaffold: proto/schema layout, codegen CI (npm + PyPI publish) | Eng Lead | `vektor-proto` |
| INFRA-002 | `vektor-platform` Turborepo + pnpm workspace init; shared package scaffolding | Eng Lead | `vektor-platform` |
| INFRA-003 | Provision K8s cluster (k3s dev + full prod) | DevOps | `vektor-infra` |
| INFRA-004 | Deploy Kafka cluster (3 brokers + Schema Registry) | DevOps | `vektor-infra` |
| INFRA-005 | Deploy PostgreSQL + PostGIS + TimescaleDB; Drizzle migrations | Backend | `vektor-platform` |
| INFRA-006 | Deploy MinIO + Qdrant + Redis Cluster | DevOps | `vektor-infra` |
| INFRA-007 | Deploy Kong API Gateway + Keycloak | DevOps | `vektor-infra` |
| INFRA-008 | CI/CD pipelines (GitHub Actions + ArgoCD) across all 6 repos | DevOps | all |
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

### Phase 2 — CV & Tracking Pipeline (Weeks 11–18)

**All tasks done as of 2026-08-23.** No GPU is available in the current sandbox, so GPU throughput itself is unverified — everything else (the real pipeline mechanics: training, export, inference, tracking, hot-swap) is verified against real infra on CPU. See [[vektor-build-conventions]] for the concrete substitutions this phase needed (`opencv4nodejs` → Kalman-filter velocity, same class of native-binding problem as `gdal-async` in Phase 1).

| Task ID | Task | Owner | Repo |
|---|---|---|---|
| ML-001 | `cv-train-svc`: YOLOv8 training + ONNX INT8 export — **done**, real end-to-end run (synthetic dataset, 1 CPU epoch, export, reload via onnxruntime) | ML | `vektor-ml` |
| ML-002 | `cv-inference-svc`: onnxruntime-node GPU build + ONNX loading — **done**; GPU path implemented with Pitfall 5's fail-fast discipline (`REQUIRE_GPU=true` never silently falls back to CPU) but unverified on real GPU hardware, which this sandbox doesn't have | ML/Backend | `vektor-platform` |
| ML-003 | `cv-inference-svc`: Kafka frame consumer → inference → detection publisher — **done**, verified against a real Kafka broker + the real checked-in ONNX fixture | Backend | `vektor-platform` |
| ML-004 | DeepSORT tracker: TypeScript re-implementation (mathjs) — **done**: Kalman filter + greedy IoU association (a documented simplification of the Hungarian algorithm) | ML/Backend | `vektor-platform` |
| ML-005 | Velocity/heading estimation — **done via the Kalman-filter path**, not optical flow: `opencv4nodejs` needs a full OpenCV source build with no prebuilt binary, not installable in this sandbox. REQ-2.5 explicitly allows either path | ML/Backend | `vektor-platform` |
| ML-006 | ONNX hot-swap endpoint — **done**, verified with two genuinely different real models (different class count *and* input size) via a real multipart upload | Backend | `vektor-platform` |

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

| Task ID | Task | Owner | Repo |
|---|---|---|---|
| ML-007 | `llm-cloud-svc`: vLLM server, LLaMA 3.1 70B, gRPC endpoint | ML | `vektor-ml` |
| ML-008 | Qdrant doctrine ingestion + BGE-M3 embeddings | ML/Backend | `vektor-platform` |
| ML-009 | `coa-svc`: context builder + Qdrant RAG retrieval | Backend | `vektor-platform` |
| ML-010 | `coa-svc`: gRPC call via `@vektor/proto` + Zod output validator | Backend | `vektor-platform` |
| ML-011 | Edge COA: node-llama-cpp + Mistral 7B Q4_K_M | Backend | `vektor-platform` |
| SVC-012 | Target Workbench threat scoring engine | Backend | `vektor-platform` |
| SVC-013 | COA approval workflow + audit-svc write | Backend | `vektor-platform` |

### Phase 5 — Logistics, Alerting & Reporting (Weeks 35–40)

| Task ID | Task | Owner |
|---|---|---|
| SVC-014 | `alert-svc`: geofence rule engine | Backend |
| SVC-015 | `alert-svc`: anomaly detection (EWMA + z-score) | Backend |
| SVC-016 | `alert-svc`: multi-channel dispatch | Backend |
| SVC-017 | `logistics-svc`: ERP CDC integration (Debezium) | Backend |
| SVC-018 | `logistics-svc`: supply forecasting (ml-regression) | Backend |
| SVC-019 | `logistics-svc`: route optimization (pgRouting) | Backend |
| SVC-020 | `reporting-svc`: PDF (Puppeteer) + PPTX (pptxgenjs) | Backend |

### Phase 6 — Edge Deployment (Weeks 41–46)

| Task ID | Task | Owner | Repo |
|---|---|---|---|
| EDGE-001 | k3s edge node config + offline bootstrap scripts | DevOps | `vektor-edge` |
| EDGE-002 | YOLOv8n INT8 ONNX build + onnxruntime-node ARM64 test on Jetson | ML | `vektor-edge` |
| EDGE-003 | node-llama-cpp Mistral 7B Q4_K_M load test on Jetson | ML | `vektor-edge` |
| EDGE-004 | Offline PMTiles map pack generation | Backend | `vektor-edge` |
| EDGE-005 | IndexedDB offline store + BullMQ sync queue | Frontend | `vektor-platform` |
| EDGE-006 | Delta sync on reconnect (Kafka offset replay) | Backend | `vektor-platform` |
| EDGE-007 | CI check: reject any edge build > 10 GB memory budget | DevOps | `vektor-edge` |

### Phase 7 — Public API, Hardening & Launch (Weeks 47–54)

| Task ID | Task | Owner |
|---|---|---|
| API-001 | Public REST API v1 + OpenAPI 3.1 spec (auto-generated from Fastify schemas) | Backend |
| API-002 | WebSocket subscription API scoped to bounding box | Backend |
| SEC-001 | Istio mTLS enforcement across all TS services | Security/DevOps |
| SEC-002 | `vektor-ml` network policy audit + gRPC-only enforcement | Security |
| SEC-003 | RBAC policy audit + row-level security (Drizzle RLS) | Security |
| SEC-004 | Penetration testing + red team exercise | Security |
| SEC-005 | Full-disk encryption (LUKS) on edge nodes | DevOps |
| PERF-001 | Load test: 10,000 events/sec sustained | QA |
| PERF-002 | onnxruntime-node benchmark: < 200ms/frame on GPU node | ML |
| PERF-003 | node-llama-cpp benchmark: < 30s COA on Jetson edge | ML |
| PERF-004 | vLLM gRPC benchmark: < 10s COA on cloud GPU | ML |
| DOC-001 | Operator training manual + runbooks | Product |
| DOC-002 | ADRs for all major architecture decisions | Engineering |

---

