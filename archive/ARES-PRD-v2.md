# ARES — Advanced Real-time Electronic Surveillance
## Product Requirements Document (PRD)

| Field | Value |
|---|---|
| **Document Version** | v2.0.0 |
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
- Real-time object detection and classification (YOLOv8 / Faster R-CNN)
- Multi-object tracking with persistent IDs (DeepSORT + optical flow)
- Multi-sensor data fusion into unified entity ontology
- Blue-force tracking with automatic no-strike zone enforcement
- COA generation via self-hosted LLM pipeline
- Target workbench with human-in-the-loop approval
- Role-based access control (RBAC)
- Web dashboard (Next.js / MapLibre / CesiumJS)
- Edge deployment mode (Jetson AGX Orin, 72h offline)

### 6.2 Out of Scope — v1.0 (Planned for v2.0)

- Autonomous strike execution (permanently out of scope — human approval required always)
- SIGINT / signals intelligence processing pipeline
- Voice command interface
- NATO federated data exchange (MIL-STD-2525)
- Simulation / wargaming mode
- Mobile native app (React Native / Flutter)

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
| REQ-1.3 | Normalize all spatial data to WGS 84 / MGRS with sub-meter precision | P0 | Spatial alignment error < 1 meter across all source types verified by test dataset |
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
| REQ-2.6 | Support custom model upload for domain-specific detection classes | P2 | Operator can upload ONNX model; system validates, deploys, and serves within 5 minutes |

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
| REQ-3.6 | Provide entity similarity search via vector embeddings (find entities matching a profile) | P2 | Similarity query returns top-10 results in < 500ms on Qdrant index |

**Canonical Entity Schema:**

```json
{
  "entity_id": "uuid-v4",
  "classification": "GroundVehicle.Tracked",
  "confidence": 0.92,
  "status": "ACTIVE | LOST | ARCHIVED | CONFLICTED",
  "affiliation": "UNKNOWN | FRIENDLY | HOSTILE | NEUTRAL",
  "source_sensors": ["drone-001", "satellite-003"],
  "position": {
    "lat": 9.0765,
    "lon": 7.3986,
    "alt_m": 320,
    "mgrs": "32NKL1234567890",
    "accuracy_m": 0.8
  },
  "kinematics": {
    "speed_kmh": 42.5,
    "heading_deg": 270,
    "trajectory": [{ "lat": ..., "lon": ..., "ts": "ISO8601" }]
  },
  "metadata": {
    "tags": ["convoy-lead", "priority-1"],
    "analyst_notes": "...",
    "no_strike": false
  },
  "first_detected": "2026-06-07T10:00:00Z",
  "last_updated": "2026-06-07T10:04:23Z"
}
```

---

### Epic 4 — Tactical AI Recommendations & Target Workbench

**Goal:** Provide operators with a prioritized target queue and AI-generated response recommendations that they can approve, modify, or reject — maintaining human in the loop at all times.

| ID | Requirement | Priority | Acceptance Criteria |
|---|---|---|---|
| REQ-4.1 | Rank active entities in Target Workbench by threat score (proximity, speed, classification, heading toward friendly assets) | P0 | Threat ranking refreshes ≤ every 5 seconds; ranking algorithm documented and auditable |
| REQ-4.2 | Generate ≥ 3 ranked COA options via self-hosted LLM, matching targets to available asset inventory | P0 | COA generated in < 10 seconds; includes title, rationale, required assets, estimated time, confidence |
| REQ-4.3 | COA interface allows operator to approve, modify, or reject with a single tap; all decisions logged with timestamp and operator ID | P0 | Approval action publishes target package within 2 seconds; full audit trail preserved |
| REQ-4.4 | COA generation must factor in: weather, terrain, available assets, no-strike zones, blue-force positions | P1 | Test suite validates COA respects no-strike zones 100% of the time |
| REQ-4.5 | Provide COA reasoning transparency — show which context factors influenced the recommendation | P1 | Operator can expand COA rationale panel; context sources listed |
| REQ-4.6 | Flag and escalate targets that remain unactioned beyond configurable time thresholds | P2 | Escalation alert fires at T+threshold; commander notified via in-app + optional webhook |

---

### Epic 5 — Alerting & Anomaly Detection

**Goal:** Proactively surface threats and anomalies before operators notice them manually.

| ID | Requirement | Priority | Acceptance Criteria |
|---|---|---|---|
| REQ-5.1 | Trigger geofence alerts when entities enter or exit defined zones | P0 | Alert fires within 3 seconds of boundary crossing |
| REQ-5.2 | Detect behavioral anomalies (unusual speed, heading change, formation breaks) via ML time-series analysis | P1 | False positive rate < 10% on validation dataset |
| REQ-5.3 | Route alerts via in-app queue, email, SMS, and configurable webhook | P1 | Alert delivered to all channels within 10 seconds of trigger |
| REQ-5.4 | Alert triage queue with priority scoring; support acknowledge, escalate, dismiss actions | P1 | All alert actions logged with operator ID and timestamp |

---

### Epic 6 — Access Control, Audit & Compliance

**Goal:** Enforce strict data access governance and maintain a complete, tamper-proof audit trail.

| ID | Requirement | Priority | Acceptance Criteria |
|---|---|---|---|
| REQ-6.1 | Implement RBAC with roles: SuperAdmin, Commander, Analyst, Logistics Officer, Field Operator, Viewer | P0 | Role matrix enforced at API level; unauthorized access returns 403 within 50ms |
| REQ-6.2 | Row-level and column-level data security based on user role and clearance | P0 | Sensitive entity fields hidden from Viewer/Field Operator roles in all API responses |
| REQ-6.3 | Immutable audit log for every data access, modification, and approval action | P0 | Audit entries written to append-only store; no update/delete permitted on audit records |
| REQ-6.4 | Session management: short-lived JWTs (15min), refresh token rotation, forced re-auth on role change | P0 | Expired token returns 401; no access granted after forced logout |

---

## 8. Non-Functional Requirements

### 8.1 Performance

| Metric | Requirement |
|---|---|
| Sensor ingest latency | < 50ms (arrival to normalized event) |
| CV inference throughput | ≥ 15 fps per active video stream |
| Object detection latency | < 200ms per frame on GPU node |
| Track fusion latency | < 500ms from detection to fused entity update |
| COA generation latency | < 10 seconds end-to-end |
| UI update latency | < 500ms from server event to rendered change |
| API response time (p99) | < 300ms for all read endpoints |
| Event ingestion throughput | ≥ 10,000 events/second sustained |

### 8.2 Availability & Reliability

| Metric | Requirement |
|---|---|
| Cloud core uptime | 99.95% (≈ 4.4 hours downtime/year) |
| Edge node uptime (offline) | 72 hours continuous operation without cloud |
| Data durability | 99.999999% (11 nines) — no data loss on node failure |
| Recovery Time Objective (RTO) | < 15 minutes on primary node failure |
| Recovery Point Objective (RPO) | < 60 seconds data loss on catastrophic failure |

### 8.3 Scalability

| Metric | Requirement |
|---|---|
| Concurrent users | ≥ 500 simultaneous operators |
| Active video streams | ≥ 50 concurrent FMV streams per cluster |
| Entity track capacity | ≥ 100,000 active tracks in PostGIS |
| Historical data | Petabyte-scale imagery archive |
| Horizontal scaling | All stateless services scale via K8s HPA without downtime |

### 8.4 Security

| Requirement | Specification |
|---|---|
| Encryption at rest | AES-256 for all stored data |
| Encryption in transit | TLS 1.3 minimum; mTLS between all internal services |
| Authentication | OAuth2/OIDC via Keycloak; MFA enforced for all roles |
| Network isolation | Zero-trust; services communicate only via service mesh (Istio) |
| Secrets management | HashiCorp Vault; no secrets in environment variables or config files |
| Vulnerability scanning | Automated SAST/DAST in CI pipeline; critical CVEs block deployment |

### 8.5 Usability

| Requirement | Specification |
|---|---|
| Device support | Renders correctly on 1024x768 minimum (tactical tablets) |
| Accessibility | WCAG 2.1 AA compliance |
| Offline UX | Offline state clearly communicated; stale data timestamp shown |
| Map performance | ≥ 60fps on WebGL map with 1,000+ active tracks |

---

## 9. System Architecture

### 9.1 Layered Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           DATA SOURCES                                  │
│   [Drones/FMV]  [SAR/Satellite]  [EW/RF]  [AIS]  [ADS-B]  [ERP/DB]    │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │  RTSP · SRT · MQTT · REST · S3 · CDC
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         INGESTION LAYER                                 │
│     FFmpeg (FMV/SRT/RTSP)  ·  GDAL (GeoTIFF/SAR)  ·  Kafka Streams    │
│                    NTP/PTP Clock Sync Daemon                            │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │ Normalized, time-aligned events
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        PROCESSING LAYER                                 │
│  ┌─────────────────────────┐   ┌───────────────────────────────────┐   │
│  │  CV Inference Engine    │   │  Stream Fusion Engine             │   │
│  │  Triton + YOLOv8        │   │  Apache Flink                     │   │
│  │  TensorRT-optimized     │   │  Extended Kalman Filter           │   │
│  │  DeepSORT Tracker       │   │  Entity dedup + ontology mapping  │   │
│  └─────────────────────────┘   └───────────────────────────────────┘   │
│  ┌─────────────────────────┐   ┌───────────────────────────────────┐   │
│  │  COA / LLM Engine       │   │  Anomaly Detection                │   │
│  │  LLaMA 3.1 / Mistral    │   │  Isolation Forest + LSTM AE       │   │
│  │  RAG via Qdrant          │   │  Rule engine (geofence, threshold)│   │
│  └─────────────────────────┘   └───────────────────────────────────┘   │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │ Enriched entities + AI outputs
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                          STORAGE LAYER                                  │
│   PostgreSQL + PostGIS  ·  TimescaleDB  ·  Qdrant  ·  MinIO  ·  Redis  │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │ REST · GraphQL · WebSocket · gRPC
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         API GATEWAY (Kong)                              │
│              OAuth2/OIDC  ·  Rate Limiting  ·  RBAC Enforcement         │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                       FRONTEND / CLIENTS                                │
│        Web Dashboard (Next.js + MapLibre GL / CesiumJS)                 │
│        Field Tablet PWA (offline-capable, 72h local state)              │
└─────────────────────────────────────────────────────────────────────────┘
```

### 9.2 Microservices Breakdown

```
ares/
├── services/
│   ├── ingest-service/           # FFmpeg/Kafka producers; NTP sync
│   ├── imagery-service/          # GeoTIFF/SAR processing via GDAL
│   ├── cv-inference-service/     # YOLOv8 via Triton; bounding box output
│   ├── tracking-service/         # DeepSORT + optical flow; persistent IDs
│   ├── fusion-service/           # Flink topology; EKF; ontology mapping
│   ├── coa-service/              # LLM pipeline; RAG; COA ranking
│   ├── alert-service/            # Rule engine; anomaly ML; notification dispatch
│   ├── auth-service/             # Keycloak integration; JWT; RBAC enforcement
│   ├── audit-service/            # Append-only audit log; lineage tracking
│   ├── geospatial-service/       # PostGIS queries; geofencing; map tiles
│   ├── logistics-service/        # ERP sync; supply forecasting; routing
│   └── reporting-service/        # PDF/PPTX sitrep generation
├── gateway/                      # Kong OSS configuration
├── frontend/
│   ├── web-dashboard/            # Next.js 15 operator console
│   └── field-pwa/                # Offline-capable PWA for field tablets
├── infra/
│   ├── k8s/                      # Kubernetes manifests + HPA configs
│   ├── terraform/                # Cloud + on-prem IaC
│   └── edge/                     # Edge node Docker Compose + offline bootstrap
└── ml/
    ├── models/                   # Model weights + TensorRT engines
    ├── training/                 # Training + fine-tuning pipelines
    └── evaluation/               # Benchmarks, confusion matrices, evals
```

### 9.3 Sensor-to-Decision Data Flow

```
[Sensor Input: FMV @ 30fps]
        │
        ▼
[FFmpeg: decode, extract frames]
        │
        ▼
[Kafka Topic: raw.video.frames]
        │
        ▼
[Triton Inference Server: YOLOv8 TensorRT]
  → Detections: {frame_id, class, bbox, confidence}
        │
        ▼
[DeepSORT Tracker]
  → Assigns persistent track_id
  → Estimates velocity + heading via optical flow
        │
        ▼
[Kafka Topic: processed.tracks]
        │
        ├──▶ [Apache Flink Fusion Engine]
        │         → Joins with EW signals, AIS, satellite data
        │         → Extended Kalman Filter for position smoothing
        │         → Deduplicates; maps to ontology schema
        │         → Writes to PostgreSQL + PostGIS
        │
        ├──▶ [Anomaly Detection Service]
        │         → Time-series scoring
        │         → Fires alerts to Alert Service
        │
        ▼
[COA Service triggered by alert or operator request]
  → Queries PostGIS (current entities, blue-force, no-strike zones)
  → Retrieves doctrine context from Qdrant (RAG)
  → Assembles prompt → LLM inference (LLaMA 3.1 70B)
  → Parses structured COA JSON
  → Returns to Target Workbench
        │
        ▼
[WebSocket push → Operator Dashboard]
[Operator: Approve / Modify / Reject]
[Approved: publishes target package → Audit log + downstream]
```

---

## 10. Tech Stack

### 10.1 Frontend

| Component | Technology | Rationale |
|---|---|---|
| Web Framework | Next.js 15 (App Router) | SSR + streaming; optimal for real-time dashboard |
| 2D Map Engine | MapLibre GL JS | Open-source, WebGL-native, vector tile support |
| 3D Map / Terrain | CesiumJS | Industry standard for 3D geospatial; CZML support |
| UI System | shadcn/ui + Tailwind CSS | Composable, accessible components; rapid iteration |
| State | Zustand + React Query | Lightweight global state + server cache sync |
| Real-time | Socket.io / native WebSocket | Live track push from fusion service |
| Video Player | Video.js + HLS.js | Drone FMV stream rendering in browser |
| Charts | Recharts | Threat timeline, sensor health graphs |
| Offline PWA | Workbox + IndexedDB | 72h offline with background sync |

### 10.2 Backend Services

| Component | Technology | Rationale |
|---|---|---|
| API Gateway | Kong OSS | Routing, auth plugins, rate limiting, logging |
| REST/GraphQL APIs | Node.js (Fastify) + TypeScript | High-concurrency, type-safe service layer |
| ML Inference APIs | Python (FastAPI) | Native PyTorch/ONNX ecosystem integration |
| Real-time Event Bus | Node.js + Socket.io | WebSocket management for 500+ clients |
| Message Broker | Apache Kafka (KRaft mode) | High-throughput, ordered, durable event streaming |
| Stream Processor | Apache Flink | Stateful, exactly-once stream processing; EKF fusion |
| Async Jobs | BullMQ (Redis-backed) | Report generation, model deployment jobs |
| Service Mesh | Istio | mTLS, observability, traffic policy enforcement |
| Identity | Keycloak | OAuth2/OIDC; MFA; RBAC token claims |

### 10.3 AI / ML

| Component | Technology | Rationale |
|---|---|---|
| Object Detection | YOLOv8 / Faster R-CNN | YOLOv8 for speed (real-time); Faster R-CNN for accuracy on SAR |
| Object Tracking | DeepSORT + ByteTrack | DeepSORT for re-ID after occlusion; ByteTrack for dense scenes |
| Inference Server | NVIDIA Triton + TensorRT | Multi-model GPU scheduling; TensorRT for max throughput |
| LLM (COA / Reasoning) | LLaMA 3.1 70B (self-hosted via vLLM) | No external data leakage; fully air-gappable |
| Embedding Model | BGE-M3 | Multilingual dense retrieval for RAG |
| Vector Store | Qdrant | High-performance ANN search; filtering support |
| Anomaly Detection | Isolation Forest + LSTM Autoencoder | Complementary: IF for point anomalies, LSTM AE for sequences |
| Forecasting | Prophet + TiDE | Logistics demand forecasting |
| Training Framework | PyTorch + Lightning | Standard, well-supported, production-ready |
| Model Versioning | MLflow | Experiment tracking, model registry |

### 10.4 Data Storage

| Store | Technology | Purpose |
|---|---|---|
| Primary Relational + Spatial | PostgreSQL 16 + PostGIS | Entity records, spatial queries, geofencing |
| Time-Series | TimescaleDB (PostgreSQL extension) | Sensor telemetry, track position history |
| Search & Logs | Elasticsearch 8 | Full-text entity search, log aggregation |
| Cache / Pub-Sub | Redis Cluster (6-node) | Hot entity cache, alert pub/sub, BullMQ |
| Object Storage | MinIO (S3-compatible) | Imagery, video clips, model weights, reports |
| Vector / Embeddings | Qdrant | RAG context retrieval, entity similarity search |
| Secrets | HashiCorp Vault | API keys, certs, DB credentials |

### 10.5 Infrastructure & DevOps

| Component | Technology | Purpose |
|---|---|---|
| Container Runtime | Docker | Service containerization |
| Orchestration | Kubernetes (K8s) | Production; k3s for edge nodes |
| IaC | Terraform | Reproducible cloud + on-prem provisioning |
| CI/CD | GitHub Actions + ArgoCD | Automated test, build, deploy pipeline |
| Monitoring | Prometheus + Grafana | Metrics, dashboards, SLA tracking |
| Logging | Loki + Grafana | Structured log aggregation |
| Tracing | Jaeger (OpenTelemetry) | Distributed request tracing |
| Alerting | Alertmanager + PagerDuty | Ops incident response |
| VPN | WireGuard | Secure edge-to-cloud tunnel |

---

## 11. Hardware Architecture & Deployment

### 11.1 Deployment Tiers

| Tier | Profile | Use Case |
|---|---|---|
| **Cloud Core** | Full K8s cluster, A100/H100 GPUs, petabyte storage | Central command; theater-level aggregation |
| **On-Premises DC** | 2-rack server room, same stack | Secure facility; no cloud dependency |
| **Tactical Edge Node** | Single ruggedized unit, Jetson AGX Orin | Forward operating posts; 72h offline |

---

### 11.2 Cloud Core / Data Center — Compute

| Node Type | Specification | Count | Purpose |
|---|---|---|---|
| GPU Inference | 2x AMD EPYC 9654 (96c), 1.5TB DDR5, 8x NVIDIA H100 PCIe 80GB | 2–4 | YOLOv8 + LLM inference |
| General Compute | 16 vCPU, 64 GB RAM | 6 | APIs, Flink, Kafka |
| Memory-Optimized | 32 vCPU, 256 GB RAM | 2 | Kafka brokers, Elasticsearch |
| Storage-Optimized | 16 vCPU, 64 GB RAM, 8 TB NVMe | 4 | TimescaleDB, MinIO |
| K8s Control Plane | 3x dual Xeon, 64 GB RAM | 3 | etcd, API server (HA) |

**Storage:**

| Type | Capacity | Technology |
|---|---|---|
| Hot imagery | 100 TB | NVMe SSD (Ceph cluster, > 10 GB/s write) |
| Cold archive | 1 PB+ | HDD (MinIO tiered storage) |
| Database volumes | 10 TB | NVMe SSD, high IOPS |
| Cache (Redis) | 512 GB | In-memory, NVMe-backed AOF |

**Network:**
- Internal backbone: **100 GbE** (dual NIC per node)
- GPU interconnect: **NVLink** (intra-node) + **InfiniBand HDR 400 Gbps** (inter-node)
- External uplink: **10 Gbps** minimum
- Internal latency: **< 1ms** pod-to-pod

---

### 11.3 On-Premises Server Rack Layout

**Rack A — Compute (42U)**

| Slots | Hardware | Spec |
|---|---|---|
| 1–4 | GPU Inference Servers | Dell PowerEdge R750xa, 2x NVIDIA H100 80GB, AMD EPYC 9654, 1.5 TB DDR5 |
| 5–8 | App / API Servers | Dell PowerEdge R650, 2x Xeon Gold 6330, 256 GB DDR4 |
| 9–12 | Stream Processing (Flink / Kafka) | Dell PowerEdge R650, 512 GB DDR4 |
| 13–15 | K8s Control Plane (HA) | 3x Dell PowerEdge R450 |
| 16–18 | Load Balancers | 2x F5 BIG-IP i4800 |
| 19–22 | Core Switches | Cisco Nexus 9300 (100 GbE) |
| 23–26 | Firewall Cluster | Palo Alto PA-5450 (HA pair) |
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

**Power & Cooling:**
- Total peak draw: **80–100 kW**
- Recommended UPS: **200 kVA** (N+1 redundancy)
- Cooling: Precision CRAC units, hot-aisle containment

---

### 11.4 Tactical Edge Node

Packaged in a single ruggedized Pelican case or vehicle-mount enclosure.

| Component | Hardware | Specification |
|---|---|---|
| **Edge AI Server** | NVIDIA Jetson AGX Orin Industrial (x2) | 64-core ARM Cortex-A78AE, 64 GB unified memory, 2048-core Ampere GPU, 275 TOPS |
| **Tactical Server** | Intel NUC 13 Pro (rugged enclosure) | 16-core i9, 64 GB RAM, 2 TB NVMe |
| **Storage** | Samsung T7 Shield SSD x4 | 4 TB total, ruggedized, AES-256 hardware encryption |
| **Network** | Peplink MAX Transit Pro | Dual 5G/LTE, Starlink (satellite) fallback, SD-WAN |
| **Switch** | Cisco IE-3400 Industrial | -40°C to 75°C, IP67 |
| **Power** | Victron MultiPlus 24V/3000W | Solar + LiPo battery + shore/generator power |
| **Display** | Xenarc 1000TSH 10" | 1000 nit sunlight-readable, IP65, glove-touch |
| **VPN** | WireGuard (hardware-accelerated) | AES-256 tunnel to cloud core |

**Edge Capabilities (Offline Mode):**
- Object detection (YOLOv8n INT8, TensorRT)
- Local map tiles (PMTiles offline pack)
- Entity state cache (last 72h from sync)
- Spot report queue (BullMQ, syncs on reconnect)
- COA via smaller local model (Mistral 7B Q4_K_M)

**Edge Model Requirements:**
- All CV models: INT8-quantized via TensorRT (< 4 GB VRAM budget)
- LLM: Q4_K_M GGUF (< 8 GB RAM on Jetson unified memory)
- Inference budget: < 150ms object detection; < 30s COA generation

---

### 11.5 End-User Devices

| Device | Spec | Assigned Role |
|---|---|---|
| Operator Workstation | Dell Precision 5860, 64 GB RAM, NVIDIA RTX 4090, dual 4K monitors | HQ commanders, analysts |
| Tactical Tablet | Getac ZX10, 10.1" FHD, Android 12, IP67 | Field supervisors |
| Ruggedized Laptop | Panasonic Toughbook 55, Intel i7, 32 GB, 14" FHD | Forward analysts |
| Field Phone | Samsung Galaxy XCover7 (IP68) | Field operators |

---

## 12. Data Architecture & Core Models

### 12.1 Data Retention Policy

| Data Type | Hot (TimescaleDB) | Warm (Elasticsearch) | Cold (MinIO) |
|---|---|---|---|
| Live entity tracks | 7 days | 90 days | 5 years |
| Raw FMV clips | 24 hours | — | 1 year |
| Imagery (satellite/SAR) | 7 days | — | 2 years |
| Sensor telemetry | 30 days | 1 year | 7 years |
| Audit logs | 90 days | 2 years | Indefinite |
| COAs & Reports | 1 year | — | Indefinite |

### 12.2 Database Index Strategy

- **PostGIS:** GIST index on all geometry columns; BRIN index on time columns
- **TimescaleDB:** Automatic chunk compression after 7 days; continuous aggregates for analytics
- **Elasticsearch:** Custom analyzers for MGRS coordinate search
- **Qdrant:** HNSW index for vector similarity; payload filters for entity type

### 12.3 Event Schema (Kafka)

All Kafka topics use **Apache Avro** with a Schema Registry.

**Topic naming convention:** `{env}.{domain}.{entity}.{action}`
Examples: `prod.ares.video.frame`, `prod.ares.entity.updated`, `prod.ares.alert.fired`

---

## 13. API Contracts

### 13.1 REST Endpoints (Core)

| Method | Endpoint | Description | Auth |
|---|---|---|---|
| `GET` | `/api/v1/entities` | List all active entities (paginated, filterable by class, affiliation, bbox) | Analyst+ |
| `GET` | `/api/v1/entities/:id` | Get full entity detail with track history | Analyst+ |
| `POST` | `/api/v1/entities/:id/tag` | Add operator tag to entity | Analyst+ |
| `GET` | `/api/v1/coa/:situation_id` | Retrieve generated COA options | Commander |
| `POST` | `/api/v1/coa/:coa_id/approve` | Approve COA option (body: `{option_rank, notes}`) | Commander |
| `POST` | `/api/v1/coa/:coa_id/reject` | Reject all COA options with reason | Commander |
| `GET` | `/api/v1/alerts` | List active alerts with priority | All roles |
| `POST` | `/api/v1/alerts/:id/ack` | Acknowledge alert | Analyst+ |
| `GET` | `/api/v1/sensors` | List all sensors with health status | All roles |
| `POST` | `/api/v1/sensors` | Register new sensor source | SuperAdmin |
| `GET` | `/api/v1/audit` | Query audit log (time range, user, action) | SuperAdmin |

### 13.2 WebSocket Events

| Event | Direction | Payload |
|---|---|---|
| `entity:updated` | Server → Client | `{ entity_id, position, kinematics, classification, ts }` |
| `entity:new` | Server → Client | Full entity schema |
| `entity:lost` | Server → Client | `{ entity_id, last_position, ts }` |
| `alert:fired` | Server → Client | `{ alert_id, type, entity_id, severity, message, ts }` |
| `coa:ready` | Server → Client | `{ coa_id, situation_id, options[] }` |
| `sensor:status` | Server → Client | `{ sensor_id, status, latency_ms, drop_rate }` |

---

## 14. Security & Compliance

### 14.1 Threat Model (Top Risks)

| Threat | Mitigation |
|---|---|
| Unauthorized data access | RBAC + row-level security + mTLS; all access logged |
| Sensor feed spoofing | Sensor authentication tokens; feed integrity checksums |
| LLM prompt injection via sensor data | Input sanitization layer before LLM context assembly |
| Edge node physical compromise | Full-disk encryption (LUKS); remote wipe capability |
| Insider threat | Principle of least privilege; anomalous access alerting |
| Supply chain (model weights) | SHA-256 checksum verification on all model downloads |

### 14.2 Compliance Targets

| Standard | Applicability |
|---|---|
| NIST SP 800-53 | US government / defense-adjacent deployments |
| ISO 27001 | International commercial deployments |
| GDPR | EU civilian / disaster response deployments |
| SOC 2 Type II | Commercial SaaS offering |

### 14.3 Zero-Trust Implementation

- Every service authenticates via mutual TLS (Istio service mesh)
- No implicit trust from network location alone
- JWTs expire every 15 minutes; refresh tokens rotate on each use
- HSM-backed key management via Vault

---

## 15. Development Roadmap

### Phase 0 — Foundation (Weeks 1–4)

**Goal:** All infrastructure and base platform running. No features yet — just bones.

| Task ID | Task | Owner | Dependencies |
|---|---|---|---|
| INFRA-001 | Provision K8s cluster (k3s dev + full prod) | DevOps | — |
| INFRA-002 | Deploy Kafka cluster (3 brokers + Schema Registry) | DevOps | INFRA-001 |
| INFRA-003 | Deploy PostgreSQL + PostGIS + TimescaleDB | DevOps | INFRA-001 |
| INFRA-004 | Deploy MinIO + Qdrant + Redis Cluster | DevOps | INFRA-001 |
| INFRA-005 | Deploy Kong API Gateway + Keycloak | DevOps | INFRA-001 |
| INFRA-006 | CI/CD pipeline (GitHub Actions + ArgoCD) | DevOps | INFRA-001 |
| INFRA-007 | Observability stack (Prometheus, Grafana, Loki, Jaeger) | DevOps | INFRA-001 |
| INFRA-008 | HashiCorp Vault + secret rotation policies | Security | INFRA-001 |

---

### Phase 1 — Ingestion & Base Map (Weeks 5–10)

**Goal:** Ingest live sensor data; display normalized tracks on map.

| Task ID | Task | Owner | Dependencies |
|---|---|---|---|
| SVC-001 | Ingest Service: RTSP/SRT → FFmpeg → Kafka producer | Backend | INFRA-002 |
| SVC-002 | Ingest Service: GeoTIFF/SAR via GDAL pipeline | Backend | INFRA-002 |
| SVC-003 | Ingest Service: AIS + ADS-B polling adapters | Backend | INFRA-002 |
| SVC-004 | NTP/PTP clock sync daemon | Backend | SVC-001 |
| SVC-005 | Geospatial Service: PostGIS CRUD + spatial query API | Backend | INFRA-003 |
| SVC-006 | Map tile server (PMTiles + vector tiles) | Backend | INFRA-003 |
| FE-001 | Web Dashboard: MapLibre GL base map + layer system | Frontend | SVC-006 |
| FE-002 | Web Dashboard: Real-time entity rendering via WebSocket | Frontend | SVC-005 |
| FE-003 | Web Dashboard: Sensor health panel | Frontend | SVC-001 |

---

### Phase 2 — CV & Tracking Pipeline (Weeks 11–18)

**Goal:** Detect, classify, and track objects from live video feeds.

| Task ID | Task | Owner | Dependencies |
|---|---|---|---|
| ML-001 | YOLOv8 model selection + TensorRT export (FP16 cloud, INT8 edge) | ML | — |
| ML-002 | Triton Inference Server deployment + model config | ML/DevOps | INFRA-001, ML-001 |
| ML-003 | CV Inference Service: frame consumer → Triton → detection publisher | ML | ML-002, SVC-001 |
| ML-004 | DeepSORT + optical flow tracker integration | ML | ML-003 |
| ML-005 | Track velocity + heading estimation (Kalman filter) | ML | ML-004 |
| FE-004 | Detection bounding box overlay on FMV player | Frontend | ML-003 |
| FE-005 | Track timeline sidebar with history replay | Frontend | ML-004 |

---

### Phase 3 — Data Fusion Engine (Weeks 19–26)

**Goal:** Merge all sensor data into unified, deduplicated entity records.

| Task ID | Task | Owner | Dependencies |
|---|---|---|---|
| SVC-007 | Apache Flink topology: stream joins across sensor topics | Backend | INFRA-002, ML-004 |
| SVC-008 | Extended Kalman Filter for position smoothing + dedup | Backend | SVC-007 |
| SVC-009 | Ontology mapping layer: detections → canonical schema | Backend | SVC-008 |
| SVC-010 | Blue-force tracking integration + no-strike zone engine | Backend | SVC-009 |
| SVC-011 | EW/RF signal correlation with CV tracks | Backend | SVC-009 |
| FE-006 | COP layer toggles (friendly/hostile/unknown/weather) | Frontend | SVC-010 |
| FE-007 | Entity detail panel with full history and source list | Frontend | SVC-009 |

---

### Phase 4 — Decision Support (Weeks 27–34)

**Goal:** COA generation, target workbench, human-in-the-loop approval.

| Task ID | Task | Owner | Dependencies |
|---|---|---|---|
| ML-006 | Self-hosted LLM deployment (LLaMA 3.1 70B via vLLM) | ML/DevOps | INFRA-001 |
| ML-007 | Qdrant doctrine ingestion + BGE-M3 embeddings | ML | INFRA-004 |
| ML-008 | COA Service: context builder + RAG retrieval | Backend | ML-006, ML-007 |
| ML-009 | COA Service: prompt chain + structured JSON output parser | Backend | ML-008 |
| SVC-012 | Target Workbench: threat scoring engine | Backend | SVC-009 |
| SVC-013 | COA approval workflow: approve/modify/reject + audit | Backend | ML-009 |
| FE-008 | Target Workbench UI: ranked threat queue | Frontend | SVC-012 |
| FE-009 | COA Panel: ranked options + rationale + approval controls | Frontend | SVC-013 |

---

### Phase 5 — Alerting, Logistics & Reporting (Weeks 35–40)

| Task ID | Task | Owner | Dependencies |
|---|---|---|---|
| SVC-014 | Alert Service: geofence rule engine | Backend | SVC-009 |
| SVC-015 | Alert Service: Isolation Forest + LSTM AE anomaly detection | ML/Backend | SVC-007 |
| SVC-016 | Alert Service: multi-channel dispatch (in-app, email, webhook) | Backend | SVC-014 |
| SVC-017 | Logistics Service: ERP CDC integration (Debezium) | Backend | INFRA-002 |
| SVC-018 | Logistics Service: Prophet-based supply forecasting | ML | SVC-017 |
| SVC-019 | Reporting Service: PDF/PPTX sitrep generation | Backend | SVC-009 |
| FE-010 | Alert triage queue UI | Frontend | SVC-016 |
| FE-011 | Logistics overlay + supply indicators | Frontend | SVC-017 |

---

### Phase 6 — Edge Deployment (Weeks 41–46)

| Task ID | Task | Owner | Dependencies |
|---|---|---|---|
| EDGE-001 | Edge k3s node configuration + offline bootstrap scripts | DevOps | INFRA-001 |
| EDGE-002 | YOLOv8n INT8 TensorRT build for Jetson AGX Orin | ML | ML-001 |
| EDGE-003 | Mistral 7B Q4_K_M local LLM for edge COA | ML | ML-006 |
| EDGE-004 | Offline map tile pack generation + PMTiles serving | Backend | SVC-006 |
| EDGE-005 | IndexedDB sync store + BullMQ offline queue | Frontend | — |
| EDGE-006 | Delta sync on reconnect (Kafka offset replay) | Backend | INFRA-002 |
| FE-012 | Field PWA: offline COP + spot report queue UI | Frontend | EDGE-005 |

---

### Phase 7 — Hardening & Launch (Weeks 47–52)

| Task ID | Task | Owner | Dependencies |
|---|---|---|---|
| SEC-001 | Istio mTLS enforcement across all services | Security/DevOps | All services |
| SEC-002 | RBAC policy audit + row-level security enforcement | Security | All services |
| SEC-003 | Penetration testing + red team exercise | Security | All |
| SEC-004 | Full-disk encryption on edge nodes (LUKS) | Security | EDGE-001 |
| PERF-001 | Load test: 10,000 events/sec ingestion sustained | QA | All |
| PERF-002 | GPU benchmark: < 200ms per frame inference | ML | ML-002 |
| PERF-003 | LLM benchmark: < 10s COA generation (cloud) | ML | ML-009 |
| DOC-001 | OpenAPI 3.1 specification (all endpoints) | Backend | All |
| DOC-002 | Operator training manual + runbooks | Product | All |
| DOC-003 | Architecture Decision Records (ADRs) | Engineering | All |

---

## 16. Risk Register

| ID | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| R-001 | GPU supply constraints delay hardware procurement | Medium | High | Order H100s 6+ months ahead; qualify AMD MI300X as fallback |
| R-002 | LLM hallucinations produce dangerous COA recommendations | Medium | Critical | Human-in-the-loop mandatory; COA cannot execute without explicit approval; confidence threshold gate |
| R-003 | Edge model too large for Jetson memory budget | High | High | INT8 quantization mandatory; strict model size budget enforced in CI |
| R-004 | Sensor timestamp skew causes phantom tracks in fusion | High | High | NTP/PTP hard requirement; GPS timestamp used — never arrival time (see Section 17) |
| R-005 | Single points of failure in Kafka or PostGIS | Medium | High | 3-broker Kafka (min ISR=2); PostgreSQL HA with Patroni; regular failover drills |
| R-006 | Slow ontology schema changes break downstream services | Medium | Medium | Schema Registry with compatibility enforcement; versioned Avro schemas |
| R-007 | vLLM/LLaMA 70B requires > available GPU VRAM | Medium | High | Quantize to Q4_K_M (fits 2x A100 80GB); or shard with tensor parallelism |
| R-008 | Key engineers depart mid-project | Low | High | Documentation culture; ADRs; pair programming; no single points of human failure |

---

## 17. Known Engineering Pitfalls

> **⚠️ PITFALL 1 — Data Synchronization Bottleneck**
>
> Do not use application-level arrival timestamps for sensor data alignment. Sensors transmit out-of-order due to variable network latency and packet drops. If the fusion engine processes events by *arrival time* rather than the embedded GPS/hardware timestamp, moving vehicles generate disjointed phantom tracking outputs — the same vehicle appears at two locations simultaneously.
>
> **Mitigation:** The Flink topology must order events by `sensor_timestamp` (GPS-derived, NTP-synced), not `kafka_received_at`. Apply a configurable event-time watermark (default: 500ms tolerance) to handle late-arriving events before windowing.

---

> **⚠️ PITFALL 2 — Edge Inference Memory Starvation**
>
> Standard FP32 object detection models will OOM-kill the Jetson AGX Orin during inference. A YOLOv8m model at FP32 requires ~4.5 GB VRAM. The Jetson shares GPU/CPU memory — running inference alongside the local database and sync service will exhaust available unified memory.
>
> **Mitigation:** All edge-deployed CV models must be INT8-quantized using TensorRT (`trtexec --int8`). Target < 1.5 GB GPU memory per model. Validate on physical hardware before field distribution. Never ship FP32 weights to edge nodes.

---

> **⚠️ PITFALL 3 — Kafka Topic Schema Drift**
>
> As services evolve, producers will change event schemas without coordinating with consumers. This silently breaks downstream services in ways that are hard to debug in production.
>
> **Mitigation:** Enforce Apache Avro schemas via Confluent Schema Registry with `BACKWARD` compatibility mode. Schema changes require a PR review from the consuming service team before merge.

---

> **⚠️ PITFALL 4 — LLM Prompt Injection via Sensor Metadata**
>
> Sensor metadata fields (names, tags, operator notes) flow directly into LLM prompts for COA generation. A crafted sensor name or analyst note could inject instructions into the LLM context.
>
> **Mitigation:** Sanitize all user-supplied and sensor-supplied strings before injecting into prompts. Use structured JSON context (not free-form text) as the primary input. Implement an output schema validator — reject any COA response that does not conform to the expected JSON schema.

---

## 18. Glossary

| Term | Definition |
|---|---|
| **ADS-B** | Automatic Dependent Surveillance–Broadcast; aircraft position transponder system |
| **AIS** | Automatic Identification System; maritime vessel tracking |
| **C2** | Command and Control |
| **COA** | Course of Action; a ranked recommended response option |
| **COP** | Common Operational Picture; unified situational awareness display |
| **CJADC2** | Combined Joint All Domain Command and Control |
| **CV** | Computer Vision |
| **DIL** | Disconnected, Intermittent, and Limited (network environment) |
| **EKF** | Extended Kalman Filter; algorithm for state estimation of nonlinear systems |
| **EW** | Electronic Warfare; RF emissions, jamming, signals intelligence |
| **FMV** | Full-Motion Video; continuous live video from drones or cameras |
| **GeoTIFF** | Georeferenced TIFF imagery format with embedded coordinate metadata |
| **HPA** | Horizontal Pod Autoscaler (Kubernetes) |
| **INT8** | 8-bit integer model quantization (vs. FP32 full precision) |
| **MGRS** | Military Grid Reference System; coordinate system used in tactical ops |
| **mAP** | Mean Average Precision; standard object detection accuracy metric |
| **mTLS** | Mutual TLS; both client and server authenticate each other |
| **NTP/PTP** | Network Time Protocol / Precision Time Protocol; clock synchronization |
| **ONNX** | Open Neural Network Exchange; model interoperability format |
| **RAG** | Retrieval-Augmented Generation; LLM technique using external knowledge retrieval |
| **RBAC** | Role-Based Access Control |
| **SAR** | Synthetic Aperture Radar; radar imaging from aircraft or satellite |
| **SRT** | Secure Reliable Transport; low-latency video streaming protocol |
| **TensorRT** | NVIDIA's inference optimization and runtime framework |
| **WGS 84** | World Geodetic System 1984; standard global coordinate reference system |

---

*ARES PRD v2.0.0 — June 2026. All specifications subject to engineering review and executive sign-off prior to implementation.*
