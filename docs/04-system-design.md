# VEKTOR — System Design & Repository Architecture

[← Back to VEKTOR-PRD.md](../VEKTOR-PRD.md)

---

## 9. System Design & Repository Architecture

### 9.1 Design Principle

The single hardest constraint in this system is the **TypeScript/Python language boundary**: 90%+ of the product must be TS for velocity and type-safety, but CV training and cloud LLM serving are Python-only ecosystems with no viable TS equivalent. Every other architectural decision in this section — repo count, contract versioning, CI topology, access control — exists to make that boundary **structurally enforced**, not just documented and hoped for.

A single monorepo *could* hold everything, including the Python enclave, but that would (a) let a TS engineer `import` Python code by accident with no repo-level guard, (b) force every commit through one CI pipeline even though GPU-heavy Python builds are 10–20x slower than TS builds, and (c) give every contributor merge access to the highest-blast-radius code (infra, model-serving) by default. Splitting into purpose-scoped repos turns each of those into a hard boundary instead of a convention.

### 9.2 Repository Map

| Repo | Contents | Primary Language | Release Cadence |
|---|---|---|---|
| **`vektor-platform`** | All TS apps (`web`, `field-pwa`), all TS microservices, shared packages (`db`, `kafka`, `ui`, `config`) | TypeScript | Continuous (multiple deploys/day) |
| **`vektor-ml`** | `cv-train-svc` (YOLOv8/Ultralytics training + ONNX export), `llm-cloud-svc` (vLLM server) | Python | Periodic (model retrain / LLM version bumps) |
| **`vektor-proto`** | Source-of-truth gRPC `.proto` + JSON-schema contracts for every cross-language and cross-service boundary; generates and publishes `@vektor/proto` (npm) and `vektor-proto` (PyPI) | Protobuf / codegen | Versioned, semver-gated |
| **`vektor-edge`** | k3s edge bootstrap, Jetson flashing/provisioning scripts, edge Docker Compose overlays, offline map-pack build tooling, hardware-in-the-loop integration tests | Shell / TS / Ansible | Tied to edge hardware release trains |
| **`vektor-infra`** | Terraform (cloud + on-prem), Kubernetes manifests, ArgoCD `Application` defs, Istio policy, Vault policy | Terraform / YAML | Change-controlled, lower frequency |
| **`vektor-docs`** | This PRD, ADRs, runbooks, threat models, glossary | Markdown | Independent of code review cadence |

### 9.3 Repo Dependency & Artifact Flow

```
                         ┌────────────────────┐
                         │   vektor-proto      │
                         │  (contracts, .proto)│
                         └──────────┬──────────┘
                     publishes versioned packages
                    ┌────────────────┴────────────────┐
                    ▼                                  ▼
        npm: @vektor/proto@x.y.z          PyPI: vektor-proto==x.y.z
                    │                                  │
                    ▼                                  ▼
        ┌───────────────────────┐          ┌───────────────────────┐
        │   vektor-platform      │          │      vektor-ml         │
        │  (TS services, web)    │          │  (cv-train, llm-cloud) │
        └───────────┬─────────────┘          └───────────┬─────────────┘
                    │ builds & pushes                     │ builds & pushes
                    ▼                                     ▼
        ┌─────────────────────────────────────────────────────────┐
        │              Container Registry (GHCR / ECR)             │
        │   ghcr.io/vektor/platform-*     ghcr.io/vektor/ml-*      │
        └───────────────────────┬───────────────────────────────────┘
                                │ image tags consumed via GitOps PR
                                ▼
                    ┌───────────────────────┐
                    │     vektor-infra       │
                    │ (Terraform, K8s, Argo) │────▶ Cloud Core / On-Prem
                    └───────────┬─────────────┘
                                │ ARM64 platform images +
                                │ ONNX/GGUF model artifacts (MinIO)
                                ▼
                    ┌───────────────────────┐
                    │      vektor-edge       │────▶ Jetson AGX Orin nodes
                    │ (k3s bootstrap, ARM64) │      (100% offline; no
                    └───────────────────────┘       runtime call to vektor-ml)
```

**Key invariant:** `vektor-platform` and `vektor-ml` never share source, a filesystem, or a network namespace at runtime. The *only* thing that crosses that boundary is (a) versioned contract packages published from `vektor-proto` at build time, and (b) gRPC calls at run time. This is the same isolation model as the "Python Enclave Security" policy in §14, just now expressed as a repo topology instead of a paragraph of prose.

### 9.4 Why a Dedicated Contract Repo (`vektor-proto`)

In the earlier ARES drafts, Zod schemas lived inside the TS monorepo (`packages/shared`) and the gRPC `.proto` files lived next to `coa-svc`. That works until the Python team needs to change a field — they either edit TS files without review context, or the contract silently drifts. Pulling contracts into their own repo makes a breaking change to the TS↔Python boundary an explicit, reviewable PR that both the platform team and the ML team must approve, with generated client stubs published as versioned packages — the same discipline gRPC/protobuf tooling gives large polyglot orgs (see `googleapis/googleapis`-style shared-proto repos).

`vektor-proto` also backs the **90-day breaking-change policy** for the public API (REQ-9.3): the public REST/WebSocket contracts are generated from the same source, so a version bump there is visible and auditable in one place.

### 9.5 Layered Runtime Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    TYPESCRIPT TERRITORY (90%+)  — vektor-platform       │
│                                                                         │
│  Ingest · Fusion · CV Serving · Tracking · COA (edge) · Auth · Audit   │
│  Alerts · Logistics · Reporting · Geospatial · Gateway · Frontend       │
│  Kafka producers/consumers · BullMQ workers · Socket.io · Redis         │
│                                                                         │
└──────────────────────────┬──────────────────────────────────────────────┘
                           │  gRPC only, via @vektor/proto — no shared FS
                ┌──────────┴──────────┐
                │   PYTHON ENCLAVE    │
                │   vektor-ml repo    │
                │  (2 services only)  │
                │                     │
                │  1. cv-train-svc    │
                │     YOLOv8 training │
                │     ONNX export     │
                │                     │
                │  2. llm-cloud-svc   │
                │     vLLM server     │
                │     LLaMA 3.1 70B   │
                └─────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           DATA SOURCES                                  │
│  [Drones/FMV] [SAR/Satellite] [EW/RF] [AIS] [ADS-B] [IoT/MQTT] [ERP]   │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │  RTSP · SRT · MQTT · REST · S3 · CDC
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    INGESTION LAYER  [TypeScript]                        │
│   fluent-ffmpeg (FMV/RTSP/SRT) · gdal-async (GeoTIFF) · kafkajs        │
│                    NTP/PTP Clock Sync                                   │
└───────────────────────────────┬─────────────────────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                   PROCESSING LAYER  [TypeScript]                        │
│  ┌──────────────────────────────┐  ┌────────────────────────────────┐  │
│  │  CV Inference Service        │  │  Stream Fusion Engine          │  │
│  │  onnxruntime-node (YOLOv8)   │  │  kafkajs + Redis Streams       │  │
│  │  GPU via CUDA bindings       │  │  Custom EKF (TypeScript)       │  │
│  │  DeepSORT tracker (TS)       │  │  BullMQ stateful workers       │  │
│  └──────────────────────────────┘  └────────────────────────────────┘  │
│  ┌──────────────────────────────┐  ┌────────────────────────────────┐  │
│  │  COA / LLM Service           │  │  Anomaly Detection             │  │
│  │  node-llama-cpp (edge)       │  │  simple-statistics (TS)        │  │
│  │  gRPC → llm-cloud-svc        │  │  Redis time-series + rules     │  │
│  └──────────────────────────────┘  └────────────────────────────────┘  │
└───────────────────────────────┬─────────────────────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                       STORAGE LAYER  [Language-agnostic]                │
│   PostgreSQL + PostGIS · TimescaleDB · Qdrant · MinIO · Redis          │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │  REST · GraphQL · WebSocket · gRPC
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      API GATEWAY  [Kong — config only]                  │
│              OAuth2/OIDC · Rate Limiting · RBAC Enforcement             │
└───────────────────────────────┬─────────────────────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    FRONTEND / CLIENTS  [TypeScript]                     │
│        Next.js 15 + MapLibre GL + CesiumJS (Web Dashboard)              │
│        Next.js PWA — offline-capable field client                       │
└─────────────────────────────────────────────────────────────────────────┘
```

### 9.6 `vektor-platform` Internal Layout (Turborepo + pnpm)

```
vektor-platform/
├── apps/
│   ├── web/                       # Next.js 15 operator dashboard
│   ├── field-pwa/                 # Next.js PWA — offline field client
│   └── api-gateway/               # Kong config + custom gateway plugins (TS)
│
├── services/
│   ├── ingest-svc/                # fluent-ffmpeg + gdal-async + kafkajs producers
│   ├── cv-inference-svc/          # onnxruntime-node YOLOv8 + DeepSORT tracker
│   ├── fusion-svc/                # Kafka consumers + Redis Streams + EKF engine
│   ├── coa-svc/                   # node-llama-cpp (edge) + gRPC client → vektor-ml
│   ├── alert-svc/                 # Rule engine + anomaly detection + BullMQ
│   ├── auth-svc/                  # Keycloak adapter + JWT + RBAC middleware
│   ├── audit-svc/                 # Append-only event store
│   ├── geospatial-svc/            # PostGIS queries + geofencing + map tile server
│   ├── logistics-svc/             # ERP sync + forecasting + pgRouting
│   └── reporting-svc/             # PDF/PPTX sitrep (puppeteer + pptxgenjs)
│
├── packages/
│   ├── shared/                    # Re-exports generated types from @vektor/proto
│   ├── db/                        # Drizzle ORM schema + migrations
│   ├── kafka/                     # kafkajs wrappers + schema helpers
│   ├── ui/                        # Shared React components (shadcn/ui base)
│   └── config/                    # ESLint, TypeScript, Prettier configs
│
└── turbo.json
```

### 9.7 Sensor-to-Decision Data Flow

```
[Sensor Input: FMV @ 30fps]
        │
        ▼
[ingest-svc: fluent-ffmpeg → extract frames as Buffer]
        │
        ▼
[kafkajs → Topic: raw.video.frames]
        │
        ▼
[cv-inference-svc: onnxruntime-node, YOLOv8 INT8 ONNX]
  → Outputs: { frame_id, class, bbox, confidence }
        │
        ▼
[DeepSORT tracker (TypeScript)]
  → Assigns persistent track_id
  → Estimates velocity + heading (optical flow via opencv4nodejs)
        │
        ▼
[kafkajs → Topic: processed.tracks]
        │
        ├──▶ [fusion-svc: Redis Streams consumer group]
        │         → EKF position smoothing (custom TS math)
        │         → Joins with EW/AIS/ADS-B streams via Redis sorted sets
        │         → Deduplicates; maps to Entity ontology
        │         → Writes to PostgreSQL + PostGIS via Drizzle ORM
        │
        ├──▶ [alert-svc: BullMQ worker]
        │         → Geofence check (PostGIS ST_Contains)
        │         → Anomaly scoring (simple-statistics z-score + EWMA)
        │         → Fires alert to Redis pub/sub
        │
        ▼
[coa-svc triggered by alert or operator request]
  → Queries PostGIS via geospatial-svc gRPC
  → RAG: Qdrant JS client → retrieves doctrine/SOP context
  → CLOUD: gRPC call (via @vektor/proto stubs) → llm-cloud-svc (vektor-ml)
  → EDGE:  node-llama-cpp local inference (Mistral 7B Q4_K_M)
  → Parses structured COA via Zod schema validator
  → Returns to Target Workbench
        │
        ▼
[Socket.io → Web Dashboard (Next.js)]
[Operator: Approve / Modify / Reject]
[Approved → audit-svc append-only log + downstream publish]
```

### 9.8 CI/CD Topology Across Repos

| Repo | CI Trigger | Pipeline | Deploy Path |
|---|---|---|---|
| `vektor-proto` | PR to contracts | Generate + typecheck TS & Python stubs; semver bump gate | Publish to npm + PyPI on merge to `main` |
| `vektor-platform` | Every PR | pnpm install → turbo lint/test/build (cached) → SAST | Build container images → push to registry on merge |
| `vektor-ml` | Every PR | Python lint/test → GPU-runner smoke test | Build container images → push to registry on merge; model artifacts → MinIO |
| `vektor-edge` | Every PR | Cross-compile ARM64 → hardware-in-the-loop tests on a physical Jetson runner (nightly, not per-PR — too slow/expensive to gate every commit) | Flash/provision images on release train |
| `vektor-infra` | Every PR | `terraform plan` posted as PR comment; policy-as-code check (OPA) | `terraform apply` requires DevOps + Security Lead approval |
| `vektor-docs` | Every PR | Markdown lint, link check | Publish on merge |

### 9.9 Repo Access Control (maps to §14 threat model)

| Repo | Default write access | Rationale |
|---|---|---|
| `vektor-platform` | All engineers | Highest velocity, lowest individual blast radius per change |
| `vektor-ml` | ML team only | GPU cost control + model-supply-chain integrity (§17, Pitfall 5-equivalent) |
| `vektor-proto` | Requires 1 platform + 1 ML approver | Forces cross-team review on any boundary change |
| `vektor-edge` | Platform + hardware/field engineers | Physical hardware access implied; kept separate from core service write access |
| `vektor-infra` | DevOps + Security Lead only | Highest blast radius in the system (§14) |
| `vektor-docs` | All engineers + Product | Lowest risk; should never block on code review cadence |

---

