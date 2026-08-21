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

| Task ID | Task | Owner |
|---|---|---|
| SVC-001 | `ingest-svc`: fluent-ffmpeg RTSP/SRT → kafkajs producer | Backend |
| SVC-002 | `ingest-svc`: gdal-async GeoTIFF/SAR pipeline | Backend |
| SVC-003 | `ingest-svc`: AIS + ADS-B + MQTT IoT adapters | Backend |
| SVC-004 | NTP/PTP clock sync daemon; `sensor_ts` validation middleware | Backend |
| SVC-005 | `geospatial-svc`: PostGIS CRUD + spatial query API | Backend |
| SVC-006 | Map tile server: PMTiles static server | Backend |
| FE-001 | Web dashboard: MapLibre GL base map + layer system | Frontend |
| FE-002 | Web dashboard: Socket.io entity rendering | Frontend |
| FE-003 | Web dashboard: sensor health panel | Frontend |

### Phase 2 — CV & Tracking Pipeline (Weeks 11–18)

| Task ID | Task | Owner | Repo |
|---|---|---|---|
| ML-001 | `cv-train-svc`: YOLOv8 training + ONNX INT8 export | ML | `vektor-ml` |
| ML-002 | `cv-inference-svc`: onnxruntime-node GPU build + ONNX loading | ML/Backend | `vektor-platform` |
| ML-003 | `cv-inference-svc`: Kafka frame consumer → inference → detection publisher | Backend | `vektor-platform` |
| ML-004 | DeepSORT tracker: TypeScript re-implementation (mathjs) | ML/Backend | `vektor-platform` |
| ML-005 | Optical flow velocity/heading estimation (opencv4nodejs) | ML/Backend | `vektor-platform` |
| ML-006 | ONNX hot-swap endpoint | Backend | `vektor-platform` |

### Phase 3 — Data Fusion Engine (Weeks 19–26)

| Task ID | Task | Owner |
|---|---|---|
| SVC-007 | `fusion-svc`: kafkajs consumer groups + Redis Streams topology | Backend |
| SVC-008 | Extended Kalman Filter (TS, mathjs) | Backend |
| SVC-009 | Ontology mapping layer: detections → canonical Entity schema (Zod) | Backend |
| SVC-010 | Blue-force tracking + no-strike zone engine (PostGIS ST_Contains) | Backend |
| SVC-011 | EW/RF signal correlation via Redis Streams join | Backend |

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

