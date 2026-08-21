# VEKTOR — Tech Stack

[← Back to VEKTOR-PRD.md](../VEKTOR-PRD.md)

---

## 10. Tech Stack

### 10.1 Frontend

| Component | Technology | Rationale |
|---|---|---|
| Web Framework | Next.js 15 (App Router) | SSR + streaming; optimal for real-time dashboard |
| 2D Map Engine | MapLibre GL JS | Open-source, WebGL-native, vector tile support |
| 3D Map / Terrain | CesiumJS | 3D geospatial; CZML support for track playback |
| UI System | shadcn/ui + Tailwind CSS | Composable, accessible; shared via `packages/ui` |
| State | Zustand + TanStack Query | Lightweight global state + server cache sync |
| Real-time | Socket.io client | Live track push; auto-reconnect |
| Video Player | Video.js + HLS.js | Drone FMV stream rendering in browser |
| Charts | Recharts | Threat timeline, sensor health graphs |
| Offline PWA | Workbox + IndexedDB (idb) | 72h offline with background sync queue |
| Type Safety | Zod | Runtime schema validation on all API responses |

### 10.2 Backend Services (All TypeScript)

| Component | Technology | Rationale |
|---|---|---|
| API Framework | Fastify + TypeScript | Fastest Node.js HTTP framework; schema validation built-in |
| API Gateway | Kong OSS | Routing, auth plugins, rate limiting, request logging |
| Real-time | Socket.io (Node.js) | WebSocket management for 500+ concurrent clients |
| Message Broker | Apache Kafka via `kafkajs` | High-throughput, ordered, durable event streaming |
| Stream Processing | `kafkajs` consumer groups + Redis Streams + BullMQ | Stateful stream processing in pure TS |
| ORM / DB | Drizzle ORM + `postgres` driver | Type-safe SQL; PostGIS + TimescaleDB compatible |
| Task Queue | BullMQ (Redis-backed) | Async jobs: report generation, model hot-swap, alerts |
| Service Mesh | Istio | mTLS, observability, traffic policy |
| Identity | Keycloak + `keycloak-js` / `keycloak-connect` | OAuth2/OIDC; MFA; RBAC JWT claims |
| gRPC | `@grpc/grpc-js` + generated `@vektor/proto` stubs | Type-safe RPC to the Python enclave |
| Validation | Zod | Schema enforcement on all inbound + LLM-outbound data |
| Logging | Pino | Structured JSON logging; minimal overhead |

### 10.3 AI / ML

| Component | Technology | Language | Rationale |
|---|---|---|---|
| **CV Model Training** | Ultralytics YOLOv8 + PyTorch | **Python** (`vektor-ml`) | No JS equivalent for training; exports ONNX for TS serving |
| **CV Model Serving** | `onnxruntime-node` (GPU build) | TypeScript | Runs exported ONNX model; CUDA-accelerated; no Python at runtime |
| **Object Tracking** | Custom DeepSORT implementation | TypeScript | Re-implemented using `mathjs` for linear algebra |
| **Cloud LLM Server** | vLLM + LLaMA 3.1 70B | **Python** (`vektor-ml`) | Best-in-class GPU throughput; exposed via gRPC only |
| **Edge LLM** | `node-llama-cpp` (Mistral 7B Q4_K_M) | TypeScript | Native Node.js LLM binding; runs on Jetson unified memory |
| **Embedding Model** | `@xenova/transformers` (BGE-M3 ONNX) | TypeScript | Transformers.js; runs BGE-M3 in Node.js |
| **Vector Store** | Qdrant JS client (`@qdrant/js-client-rest`) | TypeScript | RAG context retrieval; entity similarity search |
| **Anomaly Detection** | `simple-statistics` + custom EWMA | TypeScript | Lightweight time-series anomaly scoring |
| **Forecasting** | `ml-regression` + custom time-series | TypeScript | Logistics demand forecasting |
| **Schema Validation** | Zod | TypeScript | Validates all LLM output before it touches application logic |

### 10.4 Data Storage

| Store | Technology | TS Client | Purpose |
|---|---|---|---|
| Primary Relational + Spatial | PostgreSQL 16 + PostGIS | Drizzle ORM | Entity records, spatial queries, geofencing |
| Time-Series | TimescaleDB | Drizzle ORM | Sensor telemetry, track position history |
| Search & Logs | Elasticsearch 8 | `@elastic/elasticsearch` | Full-text entity search, log aggregation |
| Cache / Pub-Sub | Redis Cluster | `ioredis` | Hot entity cache, alert pub/sub, BullMQ backend |
| Object Storage | MinIO | `minio` (official JS SDK) | Imagery, video clips, ONNX/GGUF weights, reports |
| Vector / Embeddings | Qdrant | `@qdrant/js-client-rest` | RAG context retrieval, similarity search |
| Secrets | HashiCorp Vault | `node-vault` | API keys, TLS certs, DB credentials |

### 10.5 Infrastructure & DevOps

| Component | Technology | Purpose |
|---|---|---|
| Core monorepo tooling | Turborepo + pnpm workspaces | Unified build pipeline within `vektor-platform` |
| Container Runtime | Docker | Service containerization |
| Orchestration | Kubernetes / k3s (edge) | Production orchestration; lightweight edge runtime |
| IaC | Terraform | Reproducible cloud + on-prem provisioning (`vektor-infra`) |
| CI/CD | GitHub Actions + ArgoCD | Test, build, push, deploy pipeline, per repo (§9.8) |
| Monitoring | Prometheus + Grafana | Metrics, dashboards, SLA tracking |
| Logging | Pino → Loki → Grafana | Structured logs from all TS services |
| Tracing | OpenTelemetry JS SDK + Jaeger | Distributed tracing across all service calls |
| Service Mesh | Istio | mTLS, traffic policy, canary deploys |
| VPN | WireGuard | Secure edge-to-cloud tunnel |

---

