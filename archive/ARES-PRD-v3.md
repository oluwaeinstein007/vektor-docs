# ARES — Advanced Real-time Electronic Surveillance
## Product Requirements Document (PRD)

| Field | Value |
|---|---|
| **Document Version** | v3.0.0 |
| **Status** | In Review |
| **Classification** | Internal — Confidential |
| **Date** | June 2026 |
| **Author(s)** | Engineering Leadership |
| **Reviewers** | CTO, Product, Security Lead, ML Lead |

### Revision History

| Version | Date | Author | Summary of Changes |
|---|---|---|---|
| v1.0.0 | May 2026 | Engineering Lead | Initial draft |
| v2.0.0 | June 2026 | Engineering Lead | Full expansion: personas, NFRs, data models, API contracts, risk register, KPIs, acceptance criteria |
| v3.0.0 | June 2026 | Engineering Lead | **Language ecosystem decision:** JS/TS-first stack. Python isolated to 2 microservices only (CV training pipeline + cloud LLM server). All other services rewritten to TypeScript equivalents. Apache Flink replaced with Kafka + Redis Streams + BullMQ. vLLM replaced with node-llama-cpp at edge. New pitfall added for onnxruntime-node. Monorepo structure updated to Turborepo/pnpm. |

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Problem Statement](#2-problem-statement)
3. [Goals & Success Metrics](#3-goals--success-metrics)
4. [Stakeholders](#4-stakeholders)
5. [User Personas & Use Cases](#5-user-personas--use-cases)
6. [Scope & Boundaries](#6-scope--boundaries)
7. [Functional Requirements (Epics)](#7-functional-requirements-epics)
8. [Non-Functional Requirements](#8-non-functional-requirements)
9. [System Architecture](#9-system-architecture)
10. [Tech Stack](#10-tech-stack)
11. [Hardware Architecture & Deployment](#11-hardware-architecture--deployment)
12. [Data Architecture & Core Models](#12-data-architecture--core-models)
13. [API Contracts](#13-api-contracts)
14. [Security & Compliance](#14-security--compliance)
15. [Development Roadmap](#15-development-roadmap)
16. [Risk Register](#16-risk-register)
17. [Known Engineering Pitfalls](#17-known-engineering-pitfalls)
18. [Glossary](#18-glossary)

---

## 1. Executive Summary

**ARES (Advanced Real-time Electronic Surveillance)** is a next-generation Command and Control (C2) platform delivered as both a cloud SaaS and a ruggedized edge deployment. It ingests heterogeneous multi-domain sensor data — including Full-Motion Video (FMV), Synthetic Aperture Radar (SAR), Electronic Warfare (EW) signals, and satellite imagery — and transforms it into a single, real-time Common Operational Picture (COP).

The platform's core thesis: **data inundation kills decision speed.** Operators today switch between 8–12 disconnected tools. ARES collapses that into one fused, AI-enriched interface. Its deterministic AI recommendation engine compresses the sensor-to-decision loop from hours to under 60 seconds.

ARES is designed for civilian-grade applications including disaster response coordination, critical infrastructure monitoring, maritime surveillance, and large-scale logistics operations.

**Language Strategy:** ARES is built on a **TypeScript-first monorepo** (Turborepo + pnpm). 90%+ of the system — all APIs, stream processing, frontend, real-time, inference serving, edge runtime — is TypeScript. Python is strictly quarantined to two isolated, containerized microservices: the CV model training pipeline and the cloud-tier LLM server. All other services call these over HTTP/gRPC and have no Python dependency.

---

## 2. Problem Statement

### 2.1 Current State Pain Points

| Pain Point | Impact |
|---|---|
| Operators manage 8–12 disconnected screens and feeds | Cognitive overload; missed threats; slow decisions |
| Targeting data takes 12+ hours to propagate through workflows | Outdated picture; stale decisions |
| No correlation engine — same object reported by multiple sensors as different entities | Phantom tracks; wasted analyst effort |
| No offline capability at the tactical edge | Complete blackout when connectivity drops |
| LLM/AI tools are external, cloud-dependent, and data-leaking | Security violations; unusable in classified environments |

### 2.2 Desired Future State

A single operator console that fuses all sensor inputs in real time, automatically detects and tracks objects, correlates cross-domain data into unified entity records, and surfaces ranked, human-approved response recommendations — all deployable on-prem or at the tactical edge with zero cloud dependency.

---

## 3. Goals & Success Metrics

### 3.1 Primary Goals

| # | Goal | Metric | Target |
|---|---|---|---|
| G1 | Compress decision loop | Sensor-to-decision latency | < 60 seconds |
| G2 | Unify operator interface | Number of screens in active use | 1 (from 8–12) |
| G3 | High-confidence detection | Object detection accuracy | > 85% mAP |
| G4 | Real-time data fusion | Track deduplication rate | > 95% |
| G5 | Reliable edge operation | Uptime without cloud | 72 hours continuous |
| G6 | Fast ingestion | Sensor-to-normalization latency | < 50ms |
| G7 | Operator throughput | Targets processed per operator per hour | 10x baseline |

### 3.2 Anti-Goals (What ARES Does NOT Do)

- ARES does not autonomously execute any response action — all actions require human approval
- ARES does not replace battlefield communications systems (e.g., ATAK, LINK-16)
- ARES does not perform open-internet web browsing or external data lookups
- ARES does not store personally identifiable information (PII) of civilians
- ARES does not use any external AI API (OpenAI, Anthropic, etc.) — all inference is self-hosted

---

## 4. Stakeholders

| Role | Name / Group | Responsibility |
|---|---|---|
| Product Owner | TBD | Requirements sign-off, roadmap prioritization |
| Engineering Lead | TBD | Architecture decisions, team delivery |
| ML / CV Lead | TBD | Model selection, training pipeline, inference optimization |
| Security Lead | TBD | Threat modeling, compliance, penetration testing |
| DevOps / Platform | TBD | Infrastructure, CI/CD, edge deployments |
| End Users | Operations commanders, Analysts, Logistics officers, Field operators | Requirements input, UAT |
| Executive Sponsor | CTO | Budget, escalation, strategic alignment |

---

## 5. User Personas & Use Cases

### Persona 1 — Operations Commander

**Profile:** Senior leader at a command center. Needs an immediate, holistic situational picture. Makes high-stakes, time-sensitive decisions.

**Goals:** See everything on one screen. Get ranked recommended actions in plain language. Approve or reject with one tap.

**Frustrations:** Stale data from disconnected tools. Drowning in raw video feeds with no synthesis.

**Key Use Cases:**
- `UC-1.1` View live Common Operational Picture across all active sensors
- `UC-1.2` Review and approve/reject AI-generated Course of Action (COA)
- `UC-1.3` Declare a No-Strike Zone around blue-force assets

---

### Persona 2 — Intelligence Analyst

**Profile:** Mid-level analyst reviewing imagery, tracks, and EW data. Produces intelligence reports. Works long shifts at HQ or forward post.

**Goals:** Quickly identify, classify, and annotate objects of interest. Build pattern-of-life profiles on tracked entities.

**Frustrations:** Manual frame-by-frame video review. No persistent object IDs when targets go behind cover.

**Key Use Cases:**
- `UC-2.1` Review auto-generated detections and correct misclassifications
- `UC-2.2` Query historical track data for a specific entity
- `UC-2.3` Export annotated imagery and entity reports

---

### Persona 3 — Logistics Officer

**Profile:** Manages supply chain, asset inventory, and route planning. Needs to know what assets are available and where.

**Goals:** Real-time inventory visibility. Predictive resupply before shortfalls. Optimized routing.

**Frustrations:** ERPs disconnected from the operational picture. No forecasting capability. Route plans not updated with current threat positions.

**Key Use Cases:**
- `UC-3.1` View asset inventory levels with depletion forecasts
- `UC-3.2` Generate optimized supply route avoiding flagged threat zones
- `UC-3.3` Receive predictive low-stock alerts before depletion

---

### Persona 4 — Field Operator (Edge / Dismounted)

**Profile:** On-the-ground operator with a ruggedized tablet. Limited or no connectivity. Needs lightweight, fast interface.

**Goals:** Access current picture offline. Sync data instantly on reconnect. Simple UI that works with gloves.

**Frustrations:** Full dashboard too heavy for field hardware. Loses all data visibility when off-network.

**Key Use Cases:**
- `UC-4.1` View last-known track positions and COA offline
- `UC-4.2` Submit a spot report that queues for sync on reconnect
- `UC-4.3` Receive push alerts when connectivity restores

---

## 6. Scope & Boundaries

### 6.1 In Scope — v1.0

- Multi-source sensor ingestion (FMV, SAR, satellite, EW, AIS, ADS-B)
- Real-time object detection and classification via ONNX model served by `onnxruntime-node`
- Multi-object tracking with persistent IDs (DeepSORT logic in TypeScript)
- Multi-sensor data fusion via Kafka + Redis Streams stateful processor
- Blue-force tracking with automatic no-strike zone enforcement
- COA generation via `node-llama-cpp` (edge) and gRPC call to Python vLLM server (cloud)
- Target workbench with human-in-the-loop approval
- Role-based access control (RBAC) enforced in TypeScript API layer
- Web dashboard (Next.js / MapLibre / CesiumJS)
- Edge deployment mode (Jetson AGX Orin, k3s, 72h offline)
- Turborepo monorepo with pnpm workspace

### 6.2 Out of Scope — v1.0 (Planned for v2.0)

- Autonomous strike execution (permanently out of scope)
- SIGINT / signals intelligence processing pipeline
- Voice command interface
- NATO federated data exchange (MIL-STD-2525)
- Simulation / wargaming mode
- React Native / Flutter mobile native app

---

## 7. Functional Requirements (Epics)

> **Priority Scale:** P0 = Must ship (blocking), P1 = High value, P2 = Important, P3 = Nice to have

---

### Epic 1 — Multimodal Sensor Ingest & Normalization

**Goal:** Accept raw data from all sensor types and normalize to a canonical, time-aligned, coordinate-normalized schema before any downstream processing.

| ID | Requirement | Priority | Acceptance Criteria |
|---|---|---|---|
| REQ-1.1 | Continuously ingest FMV from drone feeds via RTSP and SRT protocols | P0 | System ingests RTSP/SRT streams at ≥ 30fps with < 50ms ingest-to-normalize latency |
| REQ-1.2 | Ingest SAR imagery and commercial satellite GeoTIFF files | P0 | GeoTIFF files processed within 5 seconds of upload; SAR imagery georeferenced correctly |
| REQ-1.3 | Normalize all spatial data to WGS 84 / MGRS with sub-meter precision | P0 | Spatial alignment error < 1 meter across all source types |
| REQ-1.4 | Synchronize all sensor streams using NTP/PTP clock alignment | P0 | Timestamp skew across sensors < 5ms after synchronization |
| REQ-1.5 | Support ingestion of AIS (maritime) and ADS-B (aircraft) tracking feeds | P1 | AIS/ADS-B objects appear on COP within 10 seconds of message receipt |
| REQ-1.6 | Expose ingestion health dashboard showing per-source status, latency, and drop rate | P1 | Dashboard reflects real-time status; alert fires if any source drops > 1% of packets |

---

### Epic 2 — Computer Vision & Automated Object Tracking

**Goal:** Run continuous AI inference on all incoming video/imagery to detect, classify, and persistently track objects of interest without manual intervention.

| ID | Requirement | Priority | Acceptance Criteria |
|---|---|---|---|
| REQ-2.1 | Run real-time CV inference on all active FMV streams simultaneously | P0 | Detection pipeline sustains ≥ 15fps inference per stream on standard GPU node |
| REQ-2.2 | Detect and classify target profiles with confidence scores ≥ 85% | P0 | mAP ≥ 0.85 on held-out validation set; false positive rate < 5% |
| REQ-2.3 | Render bounding boxes with class label and confidence score on UI | P0 | Bounding box renders within 500ms of detection event |
| REQ-2.4 | Maintain persistent track IDs across video frames, including occlusion periods up to 10 seconds | P0 | ID retention rate ≥ 90% during simulated occlusion test scenarios |
| REQ-2.5 | Implement optical flow to estimate velocity and heading of tracked objects | P1 | Velocity estimates within 10% of ground-truth in test scenarios |
| REQ-2.6 | Support custom model upload (ONNX format) for domain-specific detection classes | P2 | Operator uploads ONNX model; system validates, loads into onnxruntime-node, serves within 5 minutes |

**Supported Detection Classes (v1.0):**

```
Entity
└── Platform
    ├── GroundVehicle
    │   ├── Wheeled (civilian, utility, armored)
    │   └── Tracked (heavy equipment)
    ├── AirPlatform
    │   ├── FixedWing
    │   └── Rotary / UAV
    ├── SeaSurface
    │   ├── Vessel (commercial, patrol)
    │   └── SmallCraft
    └── Personnel
        ├── Individual
        └── Group
```

---

### Epic 3 — Data Fusion & Unified Ontology Layer

**Goal:** Correlate detections from all sensors into single, deduplicated entity records using a canonical ontology. Eliminate phantom tracks and cross-domain duplication.

| ID | Requirement | Priority | Acceptance Criteria |
|---|---|---|---|
| REQ-3.1 | Cross-reference CV tracks with EW RF emissions and satellite telemetry to build unified entity state | P0 | ≥ 95% deduplication rate validated against synthetic multi-sensor test scenarios |
| REQ-3.2 | Map all entity classifications strictly to the central domain ontology schema | P0 | 100% of output objects conform to ontology schema; schema violations logged and quarantined |
| REQ-3.3 | Enforce blue-force tracking — display friendly assets and auto-apply no-strike zones | P0 | No-strike zones render within 1 second of blue-force position update; system rejects COA targeting friendly asset |
| REQ-3.4 | Maintain full entity state history — position, velocity, classification changes, sensor source | P1 | Entity history queryable up to 30 days; time-scrub replay available in UI |
| REQ-3.5 | Detect and flag conflicting classifications across sensors for analyst review | P1 | Conflict detected within 5 seconds; queued to analyst review panel |
| REQ-3.6 | Provide entity similarity search via vector embeddings | P2 | Similarity query returns top-10 results in < 500ms on Qdrant index |

**Canonical Entity Schema:**

```typescript
// packages/shared/src/types/entity.ts
export interface Entity {
  entity_id: string;           // UUID v4
  classification: string;      // e.g. "GroundVehicle.Tracked"
  confidence: number;          // 0.0 – 1.0
  status: "ACTIVE" | "LOST" | "ARCHIVED" | "CONFLICTED";
  affiliation: "UNKNOWN" | "FRIENDLY" | "HOSTILE" | "NEUTRAL";
  source_sensors: string[];
  position: {
    lat: number;
    lon: number;
    alt_m: number;
    mgrs: string;
    accuracy_m: number;
  };
  kinematics: {
    speed_kmh: number;
    heading_deg: number;
    trajectory: Array<{ lat: number; lon: number; ts: string }>;
  };
  metadata: {
    tags: string[];
    analyst_notes: string;
    no_strike: boolean;
  };
  first_detected: string;      // ISO 8601
  last_updated: string;        // ISO 8601
}
```

---

### Epic 4 — Tactical AI Recommendations & Target Workbench

**Goal:** Provide operators with a prioritized target queue and AI-generated response recommendations that they can approve, modify, or reject — human in the loop at all times.

| ID | Requirement | Priority | Acceptance Criteria |
|---|---|---|---|
| REQ-4.1 | Rank active entities in Target Workbench by threat score | P0 | Threat ranking refreshes ≤ every 5 seconds; algorithm documented and auditable |
| REQ-4.2 | Generate ≥ 3 ranked COA options via self-hosted LLM pipeline | P0 | COA generated in < 10s (cloud); < 30s (edge); structured JSON response validated against schema |
| REQ-4.3 | COA interface: approve/modify/reject with single tap; all decisions logged | P0 | Approval action publishes target package within 2 seconds; full audit trail preserved |
| REQ-4.4 | COA generation factors in weather, terrain, assets, no-strike zones, blue-force positions | P1 | Test suite validates COA respects no-strike zones 100% of the time |
| REQ-4.5 | Show COA reasoning transparency — context factors listed per recommendation | P1 | Operator can expand COA rationale panel; context sources listed |
| REQ-4.6 | Escalate unactioned targets beyond configurable time thresholds | P2 | Escalation alert fires at T+threshold; webhook dispatched |

---

### Epic 5 — Alerting & Anomaly Detection

**Goal:** Proactively surface threats and anomalies before operators notice them manually.

| ID | Requirement | Priority | Acceptance Criteria |
|---|---|---|---|
| REQ-5.1 | Trigger geofence alerts when entities enter or exit defined zones | P0 | Alert fires within 3 seconds of boundary crossing |
| REQ-5.2 | Detect behavioral anomalies via statistical time-series analysis | P1 | False positive rate < 10% on validation dataset |
| REQ-5.3 | Route alerts via in-app queue, email, SMS, and configurable webhook | P1 | Alert delivered to all channels within 10 seconds of trigger |
| REQ-5.4 | Alert triage queue: acknowledge, escalate, dismiss; all actions logged | P1 | All alert actions logged with operator ID and timestamp |

---

### Epic 6 — Access Control, Audit & Compliance

**Goal:** Enforce strict data access governance and maintain a complete, tamper-proof audit trail.

| ID | Requirement | Priority | Acceptance Criteria |
|---|---|---|---|
| REQ-6.1 | RBAC with roles: SuperAdmin, Commander, Analyst, Logistics Officer, Field Operator, Viewer | P0 | Role matrix enforced at API level; unauthorized access returns 403 within 50ms |
| REQ-6.2 | Row-level and column-level data security based on user role | P0 | Sensitive fields hidden from Viewer/Field Operator in all API responses |
| REQ-6.3 | Immutable audit log for every access, modification, and approval action | P0 | Audit entries written to append-only store; no update/delete permitted |
| REQ-6.4 | Short-lived JWTs (15min), refresh token rotation, forced re-auth on role change | P0 | Expired token returns 401; no access granted after forced logout |

---

## 8. Non-Functional Requirements

### 8.1 Performance

| Metric | Requirement |
|---|---|
| Sensor ingest latency | < 50ms (arrival to normalized event) |
| CV inference throughput | ≥ 15 fps per stream via onnxruntime-node on GPU |
| Object detection latency | < 200ms per frame |
| Track fusion latency | < 500ms from detection to fused entity update |
| COA generation — cloud | < 10 seconds |
| COA generation — edge | < 30 seconds (node-llama-cpp, Mistral 7B Q4) |
| UI update latency | < 500ms from server event to rendered change |
| API response time (p99) | < 300ms for all read endpoints |
| Event ingestion throughput | ≥ 10,000 events/second sustained |

### 8.2 Availability & Reliability

| Metric | Requirement |
|---|---|
| Cloud core uptime | 99.95% |
| Edge node uptime (offline) | 72 hours continuous operation without cloud |
| Data durability | 99.999999% — no data loss on node failure |
| RTO | < 15 minutes on primary node failure |
| RPO | < 60 seconds data loss on catastrophic failure |

### 8.3 Scalability

| Metric | Requirement |
|---|---|
| Concurrent users | ≥ 500 simultaneous operators |
| Active video streams | ≥ 50 concurrent FMV streams per cluster |
| Entity track capacity | ≥ 100,000 active tracks in PostGIS |
| Historical data | Petabyte-scale imagery archive |
| Horizontal scaling | All stateless TS services scale via K8s HPA without downtime |

### 8.4 Security

| Requirement | Specification |
|---|---|
| Encryption at rest | AES-256 |
| Encryption in transit | TLS 1.3; mTLS between all internal services (Istio) |
| Authentication | OAuth2/OIDC via Keycloak; MFA enforced for all roles |
| Secrets management | HashiCorp Vault; no secrets in env vars or config files |
| Vulnerability scanning | Automated SAST/DAST in CI pipeline; critical CVEs block deploy |

### 8.5 Developer Experience

| Requirement | Specification |
|---|---|
| Monorepo | Turborepo + pnpm workspaces |
| Language | TypeScript strict mode across all packages |
| Shared types | `packages/shared` exports all entity schemas, Kafka event types, API contracts |
| Test coverage | ≥ 80% unit coverage; integration tests for all service boundaries |
| Local dev | `pnpm dev` spins up full stack via Docker Compose in < 2 minutes |
| Python isolation | Python services expose only HTTP/gRPC interfaces; no TS service imports Python code |

---

## 9. System Architecture

### 9.1 Language Boundary Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    TYPESCRIPT TERRITORY (90%+)                          │
│                                                                         │
│  Ingest · Fusion · CV Serving · Tracking · COA (edge) · Auth · Audit   │
│  Alerts · Logistics · Reporting · Geospatial · Gateway · Frontend       │
│  Kafka producers/consumers · BullMQ workers · Socket.io · Redis         │
│                                                                         │
└──────────────────────────┬──────────────────────────────────────────────┘
                           │  HTTP / gRPC calls only
                ┌──────────┴──────────┐
                │   PYTHON ENCLAVE    │
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

### 9.2 Layered Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           DATA SOURCES                                  │
│   [Drones/FMV]  [SAR/Satellite]  [EW/RF]  [AIS]  [ADS-B]  [ERP/DB]    │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │  RTSP · SRT · MQTT · REST · S3 · CDC
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    INGESTION LAYER  [TypeScript]                        │
│   fluent-ffmpeg (FMV/RTSP/SRT)  ·  gdal-async (GeoTIFF)  ·  kafkajs   │
│                    NTP/PTP Clock Sync (systemic-ts)                     │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │
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
│  │  gRPC → Python vLLM (cloud)  │  │  Redis time-series + rules     │  │
│  └──────────────────────────────┘  └────────────────────────────────┘  │
└───────────────────────────────┬─────────────────────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                       STORAGE LAYER  [Language-agnostic]                │
│   PostgreSQL + PostGIS  ·  TimescaleDB  ·  Qdrant  ·  MinIO  ·  Redis  │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │  REST · GraphQL · WebSocket · gRPC
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      API GATEWAY  [Kong — config only]                  │
│              OAuth2/OIDC  ·  Rate Limiting  ·  RBAC Enforcement         │
└───────────────────────────────┬─────────────────────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    FRONTEND / CLIENTS  [TypeScript]                     │
│        Next.js 15 + MapLibre GL + CesiumJS (Web Dashboard)              │
│        Next.js PWA — offline-capable field tablet client                │
└─────────────────────────────────────────────────────────────────────────┘
```

### 9.3 Monorepo Structure (Turborepo + pnpm)

```
ares/
├── apps/
│   ├── web/                       # Next.js 15 operator dashboard
│   ├── field-pwa/                 # Next.js PWA — offline field client
│   └── api-gateway/               # Kong config + custom gateway plugins (TS)
│
├── services/                      # All TypeScript microservices
│   ├── ingest-svc/                # fluent-ffmpeg + gdal-async + kafkajs producers
│   ├── cv-inference-svc/          # onnxruntime-node YOLOv8 + DeepSORT tracker
│   ├── fusion-svc/                # Kafka consumers + Redis Streams + EKF engine
│   ├── coa-svc/                   # node-llama-cpp (edge) + gRPC client → llm-cloud-svc
│   ├── alert-svc/                 # Rule engine + anomaly detection + BullMQ
│   ├── auth-svc/                  # Keycloak adapter + JWT + RBAC middleware
│   ├── audit-svc/                 # Append-only event store
│   ├── geospatial-svc/            # PostGIS queries + geofencing + map tile server
│   ├── logistics-svc/             # ERP sync + forecasting + pgRouting
│   └── reporting-svc/             # PDF/PPTX sitrep (puppeteer + pptxgenjs)
│
├── python/                        # Quarantined Python services
│   ├── cv-train-svc/              # YOLOv8 training + ONNX export (Ultralytics)
│   └── llm-cloud-svc/             # vLLM server — LLaMA 3.1 70B (cloud GPU only)
│
├── packages/                      # Shared TypeScript packages
│   ├── shared/                    # Entity types, Kafka event schemas, API contracts
│   ├── db/                        # Drizzle ORM schema + migrations
│   ├── kafka/                     # kafkajs wrappers + Avro schema helpers
│   ├── ui/                        # Shared React components (shadcn/ui base)
│   └── config/                    # ESLint, TypeScript, Prettier configs
│
├── infra/
│   ├── k8s/                       # Kubernetes manifests + HPA configs
│   ├── terraform/                 # Cloud + on-prem IaC
│   └── edge/                      # k3s edge node Docker Compose + bootstrap
│
└── turbo.json                     # Turborepo pipeline config
```

### 9.4 Sensor-to-Decision Data Flow

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
  → RAG: Qdrant JS client → retrieves doctrine context
  → CLOUD: gRPC call → llm-cloud-svc (Python vLLM, LLaMA 3.1 70B)
  → EDGE:  node-llama-cpp local inference (Mistral 7B Q4_K_M)
  → Parses structured COA via Zod schema validator
  → Returns to Target Workbench
        │
        ▼
[Socket.io → Web Dashboard (Next.js)]
[Operator: Approve / Modify / Reject]
[Approved → audit-svc append-only log + downstream publish]
```

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
| Stream Processing | `kafkajs` consumer groups + Redis Streams + BullMQ | Flink-equivalent stateful processing in pure TS |
| ORM / DB | Drizzle ORM + `postgres` driver | Type-safe SQL; PostGIS + TimescaleDB compatible |
| Task Queue | BullMQ (Redis-backed) | Async jobs: report generation, model hot-swap, alerts |
| Service Mesh | Istio | mTLS, observability, traffic policy |
| Identity | Keycloak + `keycloak-js` / `keycloak-connect` | OAuth2/OIDC; MFA; RBAC JWT claims |
| gRPC | `@grpc/grpc-js` + `@grpc/proto-loader` | Type-safe RPC between TS services + Python enclaves |
| Validation | Zod | Schema enforcement on all inbound + LLM-outbound data |
| Logging | Pino | Structured JSON logging; minimal overhead |

### 10.3 AI / ML

| Component | Technology | Language | Rationale |
|---|---|---|---|
| **CV Model Training** | Ultralytics YOLOv8 + PyTorch | **Python** (isolated) | No JS equivalent for training; exports ONNX for TS serving |
| **CV Model Serving** | `onnxruntime-node` (GPU build) | TypeScript | Runs exported ONNX model; CUDA-accelerated; no Python at runtime |
| **Object Tracking** | Custom DeepSORT implementation | TypeScript | Re-implemented in TS using `mathjs` for linear algebra |
| **Cloud LLM Server** | vLLM + LLaMA 3.1 70B | **Python** (isolated) | Best-in-class GPU throughput; exposed via gRPC only |
| **Edge LLM** | `node-llama-cpp` (Mistral 7B Q4_K_M) | TypeScript | Native Node.js LLM binding; runs on Jetson unified memory |
| **Embedding Model** | `@xenova/transformers` (BGE-M3 ONNX) | TypeScript | Hugging Face Transformers.js; runs BGE-M3 in Node.js |
| **Vector Store** | Qdrant JS client (`@qdrant/js-client-rest`) | TypeScript | RAG context retrieval; entity similarity search |
| **Anomaly Detection** | `simple-statistics` + custom EWMA | TypeScript | Lightweight time-series anomaly scoring; no Python dependency |
| **Forecasting** | `ml-regression` + custom time-series | TypeScript | Logistics demand forecasting; replaces Prophet |
| **Schema Validation** | Zod | TypeScript | Validates all LLM output before it touches application logic |

### 10.4 Data Storage

| Store | Technology | TS Client | Purpose |
|---|---|---|---|
| Primary Relational + Spatial | PostgreSQL 16 + PostGIS | Drizzle ORM | Entity records, spatial queries, geofencing |
| Time-Series | TimescaleDB | Drizzle ORM | Sensor telemetry, track position history |
| Search & Logs | Elasticsearch 8 | `@elastic/elasticsearch` | Full-text entity search, log aggregation |
| Cache / Pub-Sub | Redis Cluster | `ioredis` | Hot entity cache, alert pub/sub, BullMQ backend |
| Object Storage | MinIO | `minio` (official JS SDK) | Imagery, video clips, ONNX model weights, reports |
| Vector / Embeddings | Qdrant | `@qdrant/js-client-rest` | RAG context retrieval, similarity search |
| Secrets | HashiCorp Vault | `node-vault` | API keys, TLS certs, DB credentials |

### 10.5 Infrastructure & DevOps

| Component | Technology | Purpose |
|---|---|---|
| Monorepo | Turborepo + pnpm workspaces | Unified build pipeline; shared packages; task caching |
| Container Runtime | Docker | Service containerization |
| Orchestration | Kubernetes / k3s (edge) | Production orchestration; lightweight edge runtime |
| IaC | Terraform | Reproducible cloud + on-prem provisioning |
| CI/CD | GitHub Actions + ArgoCD | Test, build, push, deploy pipeline |
| Monitoring | Prometheus + Grafana | Metrics, dashboards, SLA tracking |
| Logging | Pino → Loki → Grafana | Structured logs from all TS services |
| Tracing | OpenTelemetry JS SDK + Jaeger | Distributed tracing across all service calls |
| Service Mesh | Istio | mTLS, traffic policy, canary deploys |
| VPN | WireGuard | Secure edge-to-cloud tunnel |

---

## 11. Hardware Architecture & Deployment

### 11.1 Deployment Tiers

| Tier | Profile | Language Runtime |
|---|---|---|
| **Cloud Core** | Full K8s cluster, H100 GPUs, petabyte storage | Node.js (TS) + isolated Python vLLM container |
| **On-Premises DC** | 2-rack server room, same stack | Node.js (TS) + isolated Python vLLM container |
| **Tactical Edge Node** | Ruggedized Jetson AGX Orin, k3s, 72h offline | Node.js (TS) only — node-llama-cpp + onnxruntime-node |

> **Edge is 100% TypeScript.** No Python runtime on edge nodes. `node-llama-cpp` and `onnxruntime-node` handle all local AI inference natively.

---

### 11.2 Cloud Core / Data Center — Compute

| Node Type | Specification | Count | Purpose |
|---|---|---|---|
| GPU — Inference + LLM | 2x AMD EPYC 9654 (96c), 1.5 TB DDR5, 8x NVIDIA H100 PCIe 80GB | 2–4 | onnxruntime-node CV + Python vLLM LLaMA 70B |
| General Compute | 16 vCPU, 64 GB RAM | 6 | Fastify APIs, Kafka, Redis, fusion workers |
| Memory-Optimized | 32 vCPU, 256 GB RAM | 2 | Kafka brokers, Elasticsearch |
| Storage-Optimized | 16 vCPU, 64 GB RAM, 8 TB NVMe | 4 | TimescaleDB, MinIO, Qdrant |
| K8s Control Plane | 3x dual Xeon, 64 GB RAM | 3 | etcd + API server (HA) |

**Storage:**

| Type | Capacity | Technology |
|---|---|---|
| Hot imagery | 100 TB | NVMe SSD (Ceph / MinIO, > 10 GB/s write) |
| Cold archive | 1 PB+ | HDD (MinIO tiered) |
| Database volumes | 10 TB | NVMe SSD, high IOPS |
| Redis cache | 512 GB | In-memory, NVMe-backed AOF |

**Network:**
- Internal backbone: **100 GbE** (dual NIC per node)
- GPU interconnect: **NVLink** + **InfiniBand HDR 400 Gbps**
- External uplink: **10 Gbps** minimum
- Pod-to-pod latency: **< 1ms**

---

### 11.3 On-Premises Server Rack Layout

**Rack A — Compute (42U)**

| Slots | Hardware | Spec |
|---|---|---|
| 1–4 | GPU Servers (CV + LLM) | Dell PowerEdge R750xa, 2x NVIDIA H100 80GB, AMD EPYC 9654, 1.5 TB DDR5 |
| 5–8 | App / API Servers (Node.js TS) | Dell PowerEdge R650, 2x Xeon Gold 6330, 256 GB DDR4 |
| 9–12 | Kafka + Redis Cluster | Dell PowerEdge R650, 512 GB DDR4 |
| 13–15 | K8s Control Plane (HA) | 3x Dell PowerEdge R450 |
| 16–18 | Load Balancers | 2x F5 BIG-IP i4800 |
| 19–22 | Core Switches | Cisco Nexus 9300 (100 GbE) |
| 23–26 | Firewall | Palo Alto PA-5450 (HA pair) |
| 27–30 | OOB Management | Dell iDRAC Enterprise |

**Rack B — Storage & Data (42U)**

| Slots | Hardware | Spec |
|---|---|---|
| 1–6 | NVMe Storage Arrays | Dell PowerStore 1200T (2x 100 TB NVMe) |
| 7–12 | HDD Cold Storage | Dell PowerVault ME5024 (2x 500 TB) |
| 13–16 | Database Servers | 4x Dell PowerEdge R740xd, 512 GB ECC RAM |
| 17–20 | Redis Cluster | 4x Supermicro, 1.5 TB DRAM total |
| 21–24 | Backup Appliances | Veeam on PowerEdge R450 |
| 25–28 | UPS + PDUs | APC Smart-UPS Ultra 20 kW x2 |

**Power & Cooling:** ~80–100 kW peak draw; 200 kVA UPS (N+1); hot-aisle containment CRAC units.

---

### 11.4 Tactical Edge Node (100% TypeScript Runtime)

| Component | Hardware | Specification |
|---|---|---|
| **Edge AI Server** | NVIDIA Jetson AGX Orin Industrial (x2) | 64-core ARM Cortex-A78AE, 64 GB unified memory, 2048-core Ampere GPU, 275 TOPS |
| **Tactical Server** | Intel NUC 13 Pro (rugged) | 16-core i9, 64 GB RAM, 2 TB NVMe |
| **Storage** | Samsung T7 Shield SSD x4 | 4 TB total, AES-256 hardware encryption |
| **Network** | Peplink MAX Transit Pro | Dual 5G/LTE, Starlink fallback, SD-WAN |
| **Switch** | Cisco IE-3400 Industrial | -40°C to 75°C, IP67 |
| **Power** | Victron MultiPlus 24V/3000W | Solar + LiPo battery + shore/generator |
| **Display** | Xenarc 1000TSH 10" | 1000 nit, IP65, glove-touch |
| **VPN** | WireGuard (hardware-accelerated) | AES-256 tunnel to cloud core |

**Edge Software Stack (Node.js / TypeScript only):**

| Component | Library | Memory Budget |
|---|---|---|
| CV Inference | `onnxruntime-node` + YOLOv8n INT8 ONNX | < 1.5 GB GPU |
| Local LLM | `node-llama-cpp` + Mistral 7B Q4_K_M | < 6 GB unified memory |
| Embeddings | `@xenova/transformers` (MiniLM ONNX) | < 256 MB |
| Offline DB | `better-sqlite3` + local PostGIS-lite queries | < 512 MB |
| Map Tiles | PMTiles static server (TS) | < 100 MB RAM |
| Sync Queue | BullMQ + `ioredis` (local Redis) | < 256 MB |
| Runtime | Node.js 22 LTS (ARM64) | — |

**Total estimated edge memory footprint: < 10 GB** (fits within Jetson AGX Orin 64 GB unified memory with headroom for OS + networking stack)

---

### 11.5 End-User Devices

| Device | Spec | Assigned Role |
|---|---|---|
| Operator Workstation | Dell Precision 5860, 64 GB RAM, NVIDIA RTX 4090, dual 4K | HQ commanders, analysts |
| Tactical Tablet | Getac ZX10, Android 12, IP67 | Field supervisors |
| Ruggedized Laptop | Panasonic Toughbook 55, i7, 32 GB | Forward analysts |
| Field Phone | Samsung Galaxy XCover7 (IP68) | Field operators |

---

## 12. Data Architecture & Core Models

### 12.1 Shared Type Package

All data schemas are defined once in `packages/shared/src/types/` and imported by every service. No schema duplication across the codebase.

```typescript
// packages/shared/src/types/index.ts
export type { Entity } from './entity'
export type { COA, COAOption } from './coa'
export type { Alert, AlertSeverity } from './alert'
export type { SensorHealth } from './sensor'
export type { KafkaEvent } from './kafka'
export type { AuditEntry } from './audit'
```

### 12.2 Drizzle ORM Schema (PostgreSQL + PostGIS)

```typescript
// packages/db/src/schema/entities.ts
import { pgTable, uuid, text, real, geometry, timestamp, jsonb } from 'drizzle-orm/pg-core'

export const entities = pgTable('entities', {
  entity_id:      uuid('entity_id').primaryKey().defaultRandom(),
  classification: text('classification').notNull(),
  confidence:     real('confidence').notNull(),
  status:         text('status').notNull().default('ACTIVE'),
  affiliation:    text('affiliation').notNull().default('UNKNOWN'),
  position:       geometry('position', { type: 'point', srid: 4326 }),
  kinematics:     jsonb('kinematics'),
  metadata:       jsonb('metadata'),
  first_detected: timestamp('first_detected').defaultNow(),
  last_updated:   timestamp('last_updated').defaultNow(),
})
```

### 12.3 Kafka Event Schemas (kafkajs + Zod)

```typescript
// packages/kafka/src/schemas/detection.ts
import { z } from 'zod'

export const DetectionEvent = z.object({
  event_id:    z.string().uuid(),
  frame_id:    z.string(),
  sensor_id:   z.string(),
  class:       z.string(),
  bbox:        z.tuple([z.number(), z.number(), z.number(), z.number()]),
  confidence:  z.number().min(0).max(1),
  sensor_ts:   z.string().datetime(),  // GPS timestamp — NEVER use kafka arrival time
  kafka_ts:    z.string().datetime(),  // For debugging only; never used for ordering
})

export type DetectionEvent = z.infer<typeof DetectionEvent>
```

**Topic naming convention:** `{env}.ares.{domain}.{action}`
Examples: `prod.ares.video.frame`, `prod.ares.entity.updated`, `prod.ares.alert.fired`

### 12.4 Data Retention Policy

| Data Type | Hot (TimescaleDB) | Warm (Elasticsearch) | Cold (MinIO) |
|---|---|---|---|
| Live entity tracks | 7 days | 90 days | 5 years |
| Raw FMV clips | 24 hours | — | 1 year |
| Imagery (satellite/SAR) | 7 days | — | 2 years |
| Sensor telemetry | 30 days | 1 year | 7 years |
| Audit logs | 90 days | 2 years | Indefinite |
| ONNX model weights | Current only (MinIO) | — | All versions |
| COAs & Reports | 1 year | — | Indefinite |

---

## 13. API Contracts

### 13.1 REST Endpoints (Fastify + TypeScript)

All endpoints are typed end-to-end. Request/response schemas defined in `packages/shared` and referenced in both Fastify route handlers and frontend API clients.

| Method | Endpoint | Description | Auth |
|---|---|---|---|
| `GET` | `/api/v1/entities` | List active entities (paginated; filter by class, affiliation, bbox) | Analyst+ |
| `GET` | `/api/v1/entities/:id` | Full entity detail with track history | Analyst+ |
| `POST` | `/api/v1/entities/:id/tag` | Add operator tag to entity | Analyst+ |
| `GET` | `/api/v1/coa/:situation_id` | Retrieve generated COA options | Commander |
| `POST` | `/api/v1/coa/:coa_id/approve` | Approve COA option `{ option_rank, notes }` | Commander |
| `POST` | `/api/v1/coa/:coa_id/reject` | Reject with reason | Commander |
| `GET` | `/api/v1/alerts` | List active alerts with priority | All roles |
| `POST` | `/api/v1/alerts/:id/ack` | Acknowledge alert | Analyst+ |
| `GET` | `/api/v1/sensors` | List sensors with health status | All roles |
| `POST` | `/api/v1/sensors` | Register new sensor source | SuperAdmin |
| `POST` | `/api/v1/models/upload` | Upload ONNX model for hot-swap | SuperAdmin |
| `GET` | `/api/v1/audit` | Query audit log (time range, user, action) | SuperAdmin |

### 13.2 WebSocket Events (Socket.io)

```typescript
// packages/shared/src/types/socket-events.ts
export interface ServerToClientEvents {
  'entity:updated': (payload: EntityUpdatedEvent) => void
  'entity:new':     (payload: Entity) => void
  'entity:lost':    (payload: { entity_id: string; last_position: Position; ts: string }) => void
  'alert:fired':    (payload: AlertEvent) => void
  'coa:ready':      (payload: { coa_id: string; situation_id: string; options: COAOption[] }) => void
  'sensor:status':  (payload: SensorHealth) => void
}

export interface ClientToServerEvents {
  'subscribe:area': (bbox: BoundingBox) => void
  'unsubscribe':    () => void
}
```

### 13.3 gRPC Interface (TS ↔ Python Boundary)

```protobuf
// proto/llm.proto — used by coa-svc (TS) to call llm-cloud-svc (Python)
syntax = "proto3";

service LLMService {
  rpc GenerateCOA (COARequest) returns (COAResponse);
}

message COARequest {
  string situation_json = 1;   // Serialized situation context
  string doctrine_context = 2; // RAG-retrieved doctrine chunks
  int32 num_options = 3;       // Always 3
}

message COAResponse {
  string coa_json = 1;         // Structured JSON; validated by Zod in TS caller
  float confidence = 2;
  int32 latency_ms = 3;
}
```

---

## 14. Security & Compliance

### 14.1 Threat Model

| Threat | Mitigation |
|---|---|
| Unauthorized data access | RBAC + row-level security + mTLS; all access logged |
| Sensor feed spoofing | Sensor auth tokens; feed integrity checksums via HMAC |
| LLM prompt injection via sensor metadata | Zod sanitization before prompt assembly; JSON-only context; output schema validation |
| Edge node physical compromise | Full-disk encryption (LUKS); remote wipe via WireGuard management channel |
| Insider threat | Principle of least privilege; anomalous access alerting via audit-svc |
| Supply chain (ONNX model weights) | SHA-256 checksum verified at load time in onnxruntime-node |
| Python enclave escape | Python containers run in isolated K8s namespace; no shared volumes with TS services; gRPC only |

### 14.2 Python Enclave Security

The two Python services are treated as **untrusted third-party dependencies** from the perspective of the TS application layer:

- Run in isolated K8s namespace (`ares-python-enclave`) with network policy restricting all traffic except gRPC from `coa-svc` and `cv-train-svc`
- No access to PostgreSQL, Redis, or MinIO directly — all storage access is proxied through TypeScript services
- Container images scanned via Trivy on every build; critical CVEs block deployment
- Resource quotas enforced (CPU, memory, GPU) to prevent runaway processes

### 14.3 Compliance Targets

| Standard | Applicability |
|---|---|
| NIST SP 800-53 | US government / defense-adjacent deployments |
| ISO 27001 | International commercial deployments |
| GDPR | EU civilian / disaster response deployments |
| SOC 2 Type II | Commercial SaaS offering |

---

## 15. Development Roadmap

### Phase 0 — Foundation & Monorepo Setup (Weeks 1–4)

**Goal:** Turborepo monorepo running, all infrastructure provisioned, shared packages scaffolded.

| Task ID | Task | Owner | Stack |
|---|---|---|---|
| INFRA-001 | Turborepo + pnpm workspace init; shared package scaffolding | Engineering Lead | TS |
| INFRA-002 | Provision K8s cluster (k3s dev + full prod) | DevOps | Terraform |
| INFRA-003 | Deploy Kafka cluster (3 brokers + Schema Registry) | DevOps | Config |
| INFRA-004 | Deploy PostgreSQL + PostGIS + TimescaleDB; Drizzle migrations | Backend | TS + SQL |
| INFRA-005 | Deploy MinIO + Qdrant + Redis Cluster | DevOps | Config |
| INFRA-006 | Deploy Kong API Gateway + Keycloak | DevOps | Config |
| INFRA-007 | CI/CD pipeline (GitHub Actions + ArgoCD) | DevOps | YAML |
| INFRA-008 | Observability stack (Prometheus, Grafana, Pino→Loki, OTel Jaeger) | DevOps | Config |
| INFRA-009 | HashiCorp Vault + secret rotation | Security | Config |
| INFRA-010 | Python enclave namespace + network policy | Security/DevOps | K8s YAML |

---

### Phase 1 — Ingestion & Base Map (Weeks 5–10)

**Goal:** Live sensor data ingested; normalized tracks rendered on map.

| Task ID | Task | Owner | Stack |
|---|---|---|---|
| SVC-001 | ingest-svc: fluent-ffmpeg RTSP/SRT → kafkajs producer | Backend | TS |
| SVC-002 | ingest-svc: gdal-async GeoTIFF/SAR pipeline | Backend | TS |
| SVC-003 | ingest-svc: AIS + ADS-B REST polling adapters | Backend | TS |
| SVC-004 | NTP/PTP clock sync daemon; sensor_ts validation middleware | Backend | TS |
| SVC-005 | geospatial-svc: PostGIS CRUD + spatial query API (Fastify + Drizzle) | Backend | TS |
| SVC-006 | Map tile server: PMTiles static server | Backend | TS |
| FE-001 | Web dashboard: MapLibre GL base map + layer system | Frontend | TS/Next.js |
| FE-002 | Web dashboard: Socket.io entity rendering | Frontend | TS/Next.js |
| FE-003 | Web dashboard: Sensor health panel | Frontend | TS/Next.js |

---

### Phase 2 — CV & Tracking Pipeline (Weeks 11–18)

**Goal:** Detect, classify, and track objects from live video.

| Task ID | Task | Owner | Stack |
|---|---|---|---|
| ML-001 | Python cv-train-svc: YOLOv8 training + ONNX INT8 export | ML | Python (enclave) |
| ML-002 | cv-inference-svc: onnxruntime-node GPU build + YOLOv8 ONNX loading | ML/Backend | TS |
| ML-003 | cv-inference-svc: Kafka frame consumer → inference → detection publisher | Backend | TS |
| ML-004 | DeepSORT tracker: TypeScript re-implementation (mathjs linear algebra) | ML/Backend | TS |
| ML-005 | Optical flow velocity + heading estimation (opencv4nodejs) | ML/Backend | TS |
| ML-006 | ONNX hot-swap endpoint: upload new model, validate, serve without restart | Backend | TS |
| FE-004 | Detection bounding box overlay on Video.js FMV player | Frontend | TS/Next.js |
| FE-005 | Track timeline sidebar with history replay | Frontend | TS/Next.js |

---

### Phase 3 — Data Fusion Engine (Weeks 19–26)

**Goal:** Unified, deduplicated entity records from all sensors.

| Task ID | Task | Owner | Stack |
|---|---|---|---|
| SVC-007 | fusion-svc: kafkajs consumer groups + Redis Streams topology | Backend | TS |
| SVC-008 | Extended Kalman Filter: TypeScript implementation for position smoothing | Backend | TS (mathjs) |
| SVC-009 | Ontology mapping layer: raw detections → canonical Entity schema (Zod) | Backend | TS |
| SVC-010 | Blue-force tracking + no-strike zone engine (PostGIS ST_Contains) | Backend | TS |
| SVC-011 | EW/RF signal correlation with CV tracks via Redis Streams join | Backend | TS |
| FE-006 | COP layer toggles (friendly/hostile/unknown/weather/logistics) | Frontend | TS/Next.js |
| FE-007 | Entity detail panel: full history, source list, time-scrub replay | Frontend | TS/Next.js |

---

### Phase 4 — Decision Support (Weeks 27–34)

**Goal:** COA generation, target workbench, human-in-the-loop approval.

| Task ID | Task | Owner | Stack |
|---|---|---|---|
| ML-007 | Python llm-cloud-svc: vLLM server, LLaMA 3.1 70B, gRPC endpoint | ML | Python (enclave) |
| ML-008 | Qdrant doctrine ingestion + @xenova/transformers BGE-M3 embeddings | ML/Backend | TS |
| ML-009 | coa-svc: context builder + Qdrant RAG retrieval | Backend | TS |
| ML-010 | coa-svc: gRPC call to llm-cloud-svc + Zod output schema validator | Backend | TS |
| ML-011 | Edge COA: node-llama-cpp + Mistral 7B Q4_K_M integration | Backend | TS |
| SVC-012 | Target Workbench: threat scoring engine (Fastify service) | Backend | TS |
| SVC-013 | COA approval workflow: approve/modify/reject + audit-svc write | Backend | TS |
| FE-008 | Target Workbench UI: ranked threat queue | Frontend | TS/Next.js |
| FE-009 | COA Panel: ranked options + rationale + approval controls | Frontend | TS/Next.js |

---

### Phase 5 — Alerting, Logistics & Reporting (Weeks 35–40)

| Task ID | Task | Owner | Stack |
|---|---|---|---|
| SVC-014 | alert-svc: geofence rule engine (PostGIS) | Backend | TS |
| SVC-015 | alert-svc: anomaly detection (simple-statistics EWMA + z-score) | Backend | TS |
| SVC-016 | alert-svc: multi-channel dispatch (in-app, email via nodemailer, webhook) | Backend | TS |
| SVC-017 | logistics-svc: ERP CDC integration (Debezium → kafkajs consumer) | Backend | TS |
| SVC-018 | logistics-svc: supply forecasting (ml-regression time-series) | Backend | TS |
| SVC-019 | reporting-svc: PDF via Puppeteer; PPTX via pptxgenjs | Backend | TS |
| FE-010 | Alert triage queue UI | Frontend | TS/Next.js |
| FE-011 | Logistics overlay + supply indicators | Frontend | TS/Next.js |

---

### Phase 6 — Edge Deployment (Weeks 41–46)

**Goal:** Full system running on Jetson with zero Python runtime.

| Task ID | Task | Owner | Stack |
|---|---|---|---|
| EDGE-001 | k3s edge node config + offline bootstrap scripts | DevOps | TS/Shell |
| EDGE-002 | YOLOv8n INT8 ONNX build + onnxruntime-node ARM64 test on Jetson | ML | TS |
| EDGE-003 | node-llama-cpp Mistral 7B Q4_K_M: load test on Jetson unified memory | ML | TS |
| EDGE-004 | Offline PMTiles map pack generation + static TS tile server | Backend | TS |
| EDGE-005 | IndexedDB (idb) offline store + BullMQ sync queue | Frontend | TS |
| EDGE-006 | Delta sync on reconnect: Kafka consumer offset replay | Backend | TS |
| EDGE-007 | Memory budget enforcement: CI check rejects any edge build > 10 GB | DevOps | TS/CI |
| FE-012 | Field PWA: offline COP + spot report queue UI | Frontend | TS/Next.js |

---

### Phase 7 — Hardening & Launch (Weeks 47–52)

| Task ID | Task | Owner | Stack |
|---|---|---|---|
| SEC-001 | Istio mTLS enforcement across all TS services | Security/DevOps | Config |
| SEC-002 | Python enclave network policy audit + gRPC-only enforcement | Security | K8s YAML |
| SEC-003 | RBAC policy audit + row-level security (Drizzle RLS policies) | Security | TS + SQL |
| SEC-004 | Penetration testing + red team exercise | Security | — |
| SEC-005 | Full-disk encryption (LUKS) on edge nodes | DevOps | Shell |
| PERF-001 | Load test: 10,000 events/sec via kafkajs sustained | QA | TS |
| PERF-002 | onnxruntime-node benchmark: < 200ms per frame on GPU node | ML | TS |
| PERF-003 | node-llama-cpp benchmark: < 30s COA on Jetson edge | ML | TS |
| PERF-004 | vLLM gRPC benchmark: < 10s COA on cloud GPU | ML | Python |
| DOC-001 | OpenAPI 3.1 specification (auto-generated from Fastify schemas) | Backend | TS |
| DOC-002 | Operator training manual + runbooks | Product | — |
| DOC-003 | Architecture Decision Records (ADRs) — especially language boundary decisions | Engineering | MD |

---

## 16. Risk Register

| ID | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| R-001 | GPU supply constraints delay hardware procurement | Medium | High | Order H100s 6+ months ahead; qualify AMD MI300X as fallback |
| R-002 | LLM hallucinations produce dangerous COA recommendations | Medium | Critical | Human-in-the-loop mandatory; Zod schema gates all LLM output; confidence threshold blocks low-quality responses |
| R-003 | onnxruntime-node GPU build fails on target hardware | High | High | Test GPU CUDA build in CI against target OS; maintain CPU fallback path; pin `onnxruntime-node` version |
| R-004 | node-llama-cpp OOM on Jetson unified memory | High | High | Q4_K_M quantization mandatory; memory budget enforced in CI; validated on physical Jetson before edge release |
| R-005 | Sensor timestamp skew causes phantom tracks | High | High | `sensor_ts` (GPS) used for all ordering — never `kafka_received_at`; watermark tolerance 500ms |
| R-006 | Kafka topic schema drift breaks consumers | Medium | Medium | Zod schemas in `packages/kafka`; schema changes require consumer team review; Avro Registry as backstop |
| R-007 | Python vLLM 70B exceeds GPU VRAM budget | Medium | High | Quantize to AWQ/GPTQ 4-bit; tensor parallel across 2x H100; fallback to 13B model |
| R-008 | DeepSORT TypeScript re-implementation diverges from reference | Medium | Medium | Unit test against reference Python outputs; maintain parity test suite in CI |
| R-009 | Single points of failure in Kafka or PostgreSQL | Medium | High | 3-broker Kafka (ISR=2); PostgreSQL HA via Patroni; weekly failover drills |
| R-010 | Key engineers depart mid-project | Low | High | ADRs for every major decision; pair programming; no single points of human failure |

---

## 17. Known Engineering Pitfalls

> **⚠️ PITFALL 1 — Data Synchronization Bottleneck**
>
> Do not use `kafka_received_at` for sensor data ordering. Sensors transmit out-of-order due to variable network latency. If the fusion engine processes events by arrival time, moving vehicles generate disjointed phantom tracks — the same vehicle appears at two locations simultaneously.
>
> **Mitigation:** The fusion-svc Kafka consumer must order events by `sensor_ts` (GPS-derived, NTP-synced). Apply a 500ms event-time watermark in the Redis Streams windowing logic to absorb late-arriving events. This is enforced by the `DetectionEvent` Zod schema which validates `sensor_ts` is a valid ISO datetime and rejects malformed events at the consumer boundary.

---

> **⚠️ PITFALL 2 — Edge Inference Memory Starvation**
>
> Standard FP32 ONNX models will OOM the Jetson AGX Orin. YOLOv8m at FP32 needs ~4.5 GB GPU memory. The Jetson has unified CPU/GPU memory — inference running alongside the local DB, BullMQ, and sync workers will exhaust the 64 GB budget.
>
> **Mitigation:** All edge ONNX models must be INT8-quantized (export via `trtexec --int8` or Ultralytics INT8 pipeline). Target < 1.5 GB per model. A CI step in the `edge` Turborepo pipeline rejects any ONNX model exceeding the size budget. Never ship FP32 weights to the edge Docker image.

---

> **⚠️ PITFALL 3 — Kafka Topic Schema Drift**
>
> As services evolve across the monorepo, Kafka producers will change event payloads without updating consumers. In a distributed system this is silent — the consumer simply receives unexpected shapes and either crashes or silently drops data.
>
> **Mitigation:** All Kafka event schemas live in `packages/kafka/src/schemas/` as Zod definitions. Every consumer validates inbound events with `DetectionEvent.parse(msg)` before processing — invalid messages go to a dead-letter topic for investigation. Schema changes require a PR that updates both the Zod schema and all consumer services simultaneously.

---

> **⚠️ PITFALL 4 — LLM Prompt Injection via Sensor Metadata**
>
> Sensor metadata (names, tags, analyst notes) flows into the COA prompt context. A crafted sensor name or operator note could inject instructions into the LLM, producing dangerous or nonsensical COA outputs.
>
> **Mitigation:** The `coa-svc` context builder sanitizes all string fields via a `sanitizeForPrompt()` utility before assembling the prompt. Context is passed as structured JSON — never as free-form text. All LLM responses are validated through a strict `COAResponse` Zod schema before touching application logic; responses failing validation are rejected and logged.

---

> **⚠️ PITFALL 5 — onnxruntime-node GPU Build Fragility**
>
> `onnxruntime-node` has a native CUDA binding that must match the exact CUDA toolkit version on the host. A mismatch between the version compiled into the npm package and the system CUDA version causes a silent load failure that falls back to CPU inference — reducing throughput by 10–50x without any obvious error.
>
> **Mitigation:** Pin `onnxruntime-node` to a specific version in `package.json`. Document the exact CUDA toolkit version it was compiled against. Add a startup health check in `cv-inference-svc` that calls `ort.env.wasm.simd` and confirms GPU execution provider is active — fail fast with a clear error if CPU-only mode is detected. Test the full Docker image against target CUDA version in CI before every release.

---

> **⚠️ PITFALL 6 — node-llama-cpp Memory Leak on Long-Running Sessions**
>
> `node-llama-cpp` maintains a native C++ context object for each inference session. On edge deployments running 24/7, failing to explicitly call `context.dispose()` and `model.dispose()` after each COA generation leads to a slow native memory leak that crashes the Node.js process after several hours.
>
> **Mitigation:** Always wrap `node-llama-cpp` calls in a `try/finally` block that disposes the context. Implement a BullMQ worker (not a long-lived in-process loop) for COA generation so each job gets a fresh context. Add a memory usage health check endpoint in `coa-svc` that alerts if RSS exceeds threshold.

---

## 18. Glossary

| Term | Definition |
|---|---|
| **ADS-B** | Automatic Dependent Surveillance–Broadcast; aircraft position transponder system |
| **AIS** | Automatic Identification System; maritime vessel tracking |
| **ONNX** | Open Neural Network Exchange; model interoperability format — the bridge between Python training and TS inference |
| **onnxruntime-node** | Node.js binding for Microsoft's ONNX Runtime; enables GPU-accelerated ML inference in TypeScript |
| **node-llama-cpp** | Node.js native binding for llama.cpp; runs quantized LLMs (Mistral, LLaMA) entirely in TypeScript |
| **@xenova/transformers** | Hugging Face Transformers.js; runs ONNX embedding models in Node.js |
| **BullMQ** | Redis-backed job queue for Node.js; used as the Flink-equivalent stateful worker system |
| **C2** | Command and Control |
| **COA** | Course of Action; a ranked recommended response option |
| **COP** | Common Operational Picture; unified situational awareness display |
| **CV** | Computer Vision |
| **DIL** | Disconnected, Intermittent, and Limited (network environment) |
| **Drizzle ORM** | TypeScript-native ORM with PostGIS and TimescaleDB support |
| **EKF** | Extended Kalman Filter; algorithm for nonlinear state estimation — re-implemented in TypeScript using mathjs |
| **EW** | Electronic Warfare; RF emissions, jamming, signals intelligence |
| **FMV** | Full-Motion Video; continuous live video from drones or cameras |
| **GeoTIFF** | Georeferenced TIFF imagery format with embedded coordinate metadata |
| **gRPC** | Google Remote Procedure Call; the only communication protocol between TypeScript services and Python enclaves |
| **HPA** | Horizontal Pod Autoscaler (Kubernetes) |
| **INT8** | 8-bit integer quantization; mandatory for all edge-deployed ONNX models |
| **kafkajs** | Pure TypeScript Apache Kafka client |
| **MGRS** | Military Grid Reference System; tactical coordinate system |
| **mAP** | Mean Average Precision; object detection accuracy metric |
| **mTLS** | Mutual TLS; both client and server authenticate |
| **NTP/PTP** | Network Time Protocol / Precision Time Protocol; clock sync for sensor alignment |
| **RAG** | Retrieval-Augmented Generation; LLM with external knowledge retrieval via Qdrant |
| **RBAC** | Role-Based Access Control |
| **SAR** | Synthetic Aperture Radar |
| **SRT** | Secure Reliable Transport; low-latency video streaming protocol |
| **Turborepo** | Monorepo build system; manages task pipelines across all ARES packages and services |
| **vLLM** | Python library for high-throughput LLM serving; used only in the Python cloud LLM enclave |
| **WGS 84** | World Geodetic System 1984; global coordinate reference system |
| **Zod** | TypeScript-first schema validation library; enforces type safety at all service boundaries and LLM outputs |

---

*ARES PRD v3.0.0 — June 2026. TypeScript-first implementation. Python strictly isolated to cv-train-svc and llm-cloud-svc. All specifications subject to engineering review and executive sign-off prior to implementation.*
