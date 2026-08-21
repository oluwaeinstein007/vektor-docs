# Maven Smart System (MSS) Clone — Full Build Document

> **Codename:** ATLAS — AI-Enabled Tactical & Logistics Awareness System  
> **Version:** 1.0.0  
> **Classification:** Unclassified / Civilian Research Build  
> **Date:** June 2026

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Product Requirements Document (PRD)](#2-product-requirements-document-prd)
3. [System Architecture](#3-system-architecture)
4. [Feature Specifications](#4-feature-specifications)
5. [Tech Stack](#5-tech-stack)
6. [Hardware & Infrastructure Requirements](#6-hardware--infrastructure-requirements)
7. [AI/ML Pipeline Design](#7-aiml-pipeline-design)
8. [Data Architecture](#8-data-architecture)
9. [Security & Compliance](#9-security--compliance)
10. [Development Task Breakdown](#10-development-task-breakdown)
11. [Milestones & Roadmap](#11-milestones--roadmap)

---

## 1. Executive Summary

ATLAS is a civilian-grade, AI-powered command and control (C2) situational awareness platform inspired by the US DoD's Maven Smart System. It is designed to unify disparate data feeds (satellite imagery, IoT sensors, drone feeds, weather, logistics) into a single fused visualization interface, enabling real-time multi-domain decision support.

**Target Use Cases (Civilian/Commercial):**
- Disaster response coordination (FEMA-style command centers)
- Border & maritime surveillance
- Critical infrastructure monitoring (power grids, pipelines)
- Large-scale logistics and supply chain management
- Smart city operations centers

---

## 2. Product Requirements Document (PRD)

### 2.1 Goals

| Goal | Description |
|------|-------------|
| **Data Fusion** | Ingest and normalize data from 10+ heterogeneous sources in real time |
| **AI Targeting** | Automatically detect, classify, and track objects of interest using computer vision |
| **Decision Support** | Provide ranked recommended actions with confidence scores |
| **Speed** | Compress sensor-to-decision timelines from hours to under 60 seconds |
| **Interoperability** | Open extensible API for third-party integrations |
| **Resilience** | Operate in degraded/disconnected environments (edge deployment) |

### 2.2 User Personas

**1. Commander / Operations Lead**
- Needs: Single-pane situational picture, fast COA (Course of Action) generation
- Pain Points: Switching between 8–10 separate tools, stale data, delayed reports

**2. Intelligence Analyst**
- Needs: Imagery analysis tools, object tracking, pattern-of-life analysis
- Pain Points: Manual image review, annotation bottlenecks

**3. Logistics Officer**
- Needs: Supply forecasting, route optimization, inventory visibility
- Pain Points: Disconnected ERPs, no predictive capability

**4. Field Operator (Edge)**
- Needs: Lightweight mobile interface, offline capability, fast sync on reconnect
- Pain Points: No connectivity, heavy client software

### 2.3 Non-Functional Requirements

| Requirement | Specification |
|---|---|
| Latency | < 500ms UI response; < 2s for AI inference on standard object detection |
| Availability | 99.95% uptime (HA deployment) |
| Throughput | 10,000+ events/second ingestion |
| Data Volume | Support petabyte-scale historical imagery storage |
| Concurrent Users | 500+ simultaneous operators |
| Encryption | AES-256 at rest, TLS 1.3 in transit |
| Offline Mode | Full core functionality for 72 hours without cloud connectivity |

### 2.4 Constraints

- No use of third-party AI APIs (all inference on-premise or private cloud)
- All data pipelines must be auditable with full lineage tracking
- Role-based access control (RBAC) enforced at row/column level
- UI must render on tactical tablets (1024x768 minimum resolution)

---

## 3. System Architecture

### 3.1 High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         DATA SOURCES                                │
│  [Drones/UAVs] [Satellites] [IoT Sensors] [Weather APIs] [ERP/DB]  │
└────────────────────────┬────────────────────────────────────────────┘
                         │ Raw Streams (RTSP, REST, MQTT, S3)
                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     INGESTION LAYER                                 │
│        Apache Kafka  ·  Apache Flink  ·  Debezium CDC              │
└────────────────────────┬────────────────────────────────────────────┘
                         │ Normalized Event Streams
                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    AI/ML PROCESSING LAYER                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │
│  │ Object Detect│  │  Tracking    │  │  NLP / LLM Reasoning     │  │
│  │ (YOLO v10)   │  │  (DeepSORT)  │  │  (LLaMA 3 / Mistral)     │  │
│  └──────────────┘  └──────────────┘  └──────────────────────────┘  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │
│  │ Logistics    │  │  Anomaly     │  │  COA Generator           │  │
│  │ Forecasting  │  │  Detection   │  │  (RAG + LLM)             │  │
│  └──────────────┘  └──────────────┘  └──────────────────────────┘  │
└────────────────────────┬────────────────────────────────────────────┘
                         │ Enriched Events + AI Outputs
                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      DATA STORE LAYER                               │
│  [TimescaleDB] [PostGIS] [Elasticsearch] [MinIO] [Redis] [Weaviate] │
└────────────────────────┬────────────────────────────────────────────┘
                         │ REST / GraphQL / WebSocket / gRPC
                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        API GATEWAY                                  │
│              Kong Gateway  ·  OAuth2/OIDC  ·  Rate Limiting         │
└────────────────────────┬────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     FRONTEND / CLIENTS                              │
│  [Web Dashboard]  [Tactical Tablet App]  [Edge Mobile Client]       │
│   (Next.js / MapLibre GL)  (React Native)   (Flutter)               │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 Microservices Architecture

```
atlas/
├── services/
│   ├── ingest-service/           # Kafka producers for all data sources
│   ├── imagery-service/          # Satellite/drone image processing pipeline
│   ├── object-detection-service/ # YOLO-based CV inference
│   ├── tracking-service/         # Multi-object tracking (DeepSORT)
│   ├── fusion-service/           # Multi-source data correlation engine
│   ├── logistics-service/        # Supply chain forecasting & routing
│   ├── coa-service/              # Course of Action generation (LLM-powered)
│   ├── alert-service/            # Anomaly detection + notification dispatch
│   ├── auth-service/             # Identity, RBAC, JWT/OAuth2
│   ├── audit-service/            # Full audit trail / lineage tracking
│   ├── geospatial-service/       # Geofencing, spatial queries, map tiles
│   └── reporting-service/        # Report generation, PDF/PPTX export
├── gateway/                      # Kong API Gateway config
├── frontend/
│   ├── web-dashboard/            # Next.js operator console
│   ├── mobile-tablet/            # React Native tactical app
│   └── edge-client/              # Flutter offline-first field app
├── infra/
│   ├── k8s/                      # Kubernetes manifests
│   ├── terraform/                # IaC for cloud/on-prem
│   └── docker-compose/           # Local/edge deployment
└── ml/
    ├── models/                   # Model weights, configs
    ├── training/                 # Training pipelines
    └── evaluation/               # Benchmarks and evals
```

### 3.3 Data Flow — Sensor to Decision

```
[Sensor Input]
     │
     ▼
[Kafka Topic: raw.sensor.events]
     │
     ▼
[Flink Stream Processor]
  → Normalize schema
  → Deduplicate
  → Enrich with metadata
     │
     ▼
[Kafka Topic: processed.events]
     │
     ├──▶ [Object Detection Service]
     │         → YOLO inference
     │         → Bounding boxes + class labels
     │         → Publishes to: kafka/detections
     │
     ├──▶ [Tracking Service]
     │         → DeepSORT tracker
     │         → Assigns persistent IDs
     │         → Velocity + trajectory estimation
     │
     ├──▶ [Fusion Service]
     │         → Correlates detections across sensors
     │         → Resolves duplicate observations
     │         → Outputs unified tracks
     │
     ▼
[PostGIS + TimescaleDB]
  → Spatial indexing
  → Time-series storage
     │
     ▼
[COA Service]
  → Retrieves context from vector DB (Weaviate)
  → Generates 3 ranked courses of action via LLM
  → Returns with confidence + rationale
     │
     ▼
[WebSocket Push → Frontend Dashboard]
```

---

## 4. Feature Specifications

### 4.1 Core Features

#### F1 — Unified Situational Picture (Common Operating Picture)
- Real-time map with all tracks, sensors, and overlays
- Layer toggles: friendly/unknown/threat/weather/logistics
- Time-scrubbing (replay any past state)
- Heat maps for activity density

#### F2 — Multi-Source Data Ingestion
- Drone / UAV live video (RTSP stream)
- Satellite imagery (manual upload + automated S3 triggers)
- IoT sensor telemetry (MQTT broker)
- Weather data (NOAA API / OpenWeatherMap)
- AIS (ship tracking) / ADS-B (aircraft tracking)
- ERP/logistics DB sync via CDC (Debezium)

#### F3 — AI Object Detection & Classification
- Real-time YOLO v10 inference on video/imagery
- Supported classes: vehicles, personnel, vessels, aircraft, structures
- Confidence scores per detection
- Custom class training support

#### F4 — Multi-Object Tracking
- Persistent object IDs across frames and sensor views
- Trajectory prediction (Kalman filter)
- Pattern-of-life analysis (behavioral baseline)
- Dwell time alerts

#### F5 — Course of Action (COA) Generator
- LLM-powered analysis of current situation
- Generates 3 ranked COAs with pros/cons
- Factors in: available assets, terrain, weather, logistics state
- Commander can accept, modify, or reject

#### F6 — Logistics Intelligence
- Supply level monitoring with depletion rate forecasting
- Route optimization (A* / Dijkstra on road graph)
- Predictive resupply scheduling
- Bottleneck and risk flagging

#### F7 — Alerting & Anomaly Detection
- Rule-based triggers (geofence breach, threshold crossing)
- ML-based anomaly detection on time-series data
- Multi-channel alerts: in-app, email, SMS, Slack webhook
- Alert triage queue with priority scoring

#### F8 — Reporting & Export
- Auto-generated PDF/PPTX situation reports
- Custom report builder with drag-and-drop widgets
- Scheduled report dispatch
- Raw data export (CSV, GeoJSON, Shapefile)

#### F9 — Role-Based Access Control (RBAC)
- Roles: SuperAdmin, Commander, Analyst, Logistics Officer, Field Operator, Viewer
- Row-level and column-level data security
- Attribute-based policies (ABAC extension)
- Full audit log of every access/action

#### F10 — Edge / Offline Mode
- PWA + local SQLite sync for web
- Flutter client with background sync queue
- Conflict resolution on reconnect (CRDT-based)
- Operates for 72h without network

### 4.2 Advanced / Phase 2 Features

| Feature | Description |
|---|---|
| **Multi-domain fusion** | Correlate air, land, sea, space, cyber tracks |
| **Simulation mode** | Run "what-if" scenarios without affecting live data |
| **AI model marketplace** | Plug in custom detection models per deployment |
| **NATO/allied data sharing** | Federated data exchange via standard protocols (MIL-STD-2525) |
| **Voice command interface** | LLM-powered natural language commands |
| **Automated SIGINT analysis** | Signal intelligence processing pipeline |

---

## 5. Tech Stack

### 5.1 Frontend

| Layer | Technology | Purpose |
|---|---|---|
| Web Framework | Next.js 15 (App Router) | Main operator dashboard |
| Map Engine | MapLibre GL JS | 2D/3D geospatial visualization |
| 3D Globe | Deck.gl / Cesium.js | 3D situational picture |
| UI Components | shadcn/ui + Tailwind CSS | Design system |
| State Management | Zustand + React Query | App state + server sync |
| Real-time | Socket.io client / WebSocket | Live track updates |
| Mobile (Tablet) | React Native + Mapbox SDK | Tactical tablet app |
| Edge Client | Flutter | Offline-first field app |
| Charts | Recharts / Victory Native | Analytics dashboards |
| Video Player | Video.js / HLS.js | Drone feed streaming |

### 5.2 Backend Services

| Service | Technology | Purpose |
|---|---|---|
| API Gateway | Kong OSS | Routing, auth, rate limiting |
| Core APIs | Node.js (Fastify) | REST/GraphQL services |
| ML Services | Python (FastAPI) | AI inference endpoints |
| Real-time | Node.js + Socket.io | WebSocket event bus |
| Message Queue | Apache Kafka | High-throughput event streaming |
| Stream Processing | Apache Flink | Real-time stream analytics |
| Task Queue | BullMQ (Redis-backed) | Async job processing |
| Service Mesh | Istio | mTLS, observability, traffic mgmt |

### 5.3 AI / ML

| Component | Technology | Purpose |
|---|---|---|
| Object Detection | YOLOv10 / RT-DETR | Real-time imagery detection |
| Object Tracking | DeepSORT / ByteTrack | Multi-object tracking |
| LLM (COA/Reasoning) | LLaMA 3.1 70B / Mistral 7B | Language reasoning, COA gen |
| Embedding Model | BGE-M3 | Semantic search, RAG |
| Vector DB | Weaviate | Embedding storage for RAG |
| Anomaly Detection | Isolation Forest / LSTM AE | Time-series anomaly detection |
| Forecasting | Prophet / TiDE | Logistics demand forecasting |
| Training Framework | PyTorch + Lightning | Model training pipelines |
| Model Serving | Triton Inference Server | GPU-accelerated inference |
| Experiment Tracking | MLflow | Model versioning + metrics |
| Feature Store | Feast | ML feature management |

### 5.4 Data Storage

| Store | Technology | Purpose |
|---|---|---|
| Primary DB | PostgreSQL 16 + PostGIS | Relational + geospatial data |
| Time-series | TimescaleDB | Sensor/telemetry time-series |
| Search | Elasticsearch 8 | Full-text + log search |
| Cache | Redis Cluster | Session, hot data, pub/sub |
| Object Storage | MinIO (S3-compatible) | Imagery, video, model files |
| Vector DB | Weaviate | RAG context embeddings |
| Graph DB | Neo4j | Asset relationships, logistics graph |
| Message Log | Apache Kafka (persistent) | Event sourcing + audit |

### 5.5 Infrastructure & DevOps

| Component | Technology | Purpose |
|---|---|---|
| Container Runtime | Docker | Containerization |
| Orchestration | Kubernetes (K8s) | Container orchestration |
| IaC | Terraform | Infrastructure as code |
| CI/CD | GitHub Actions + ArgoCD | Automated deploy pipeline |
| Service Mesh | Istio | Secure inter-service comms |
| Secrets | HashiCorp Vault | Secrets management |
| Monitoring | Prometheus + Grafana | Metrics and dashboards |
| Logging | Loki + Grafana | Centralized log aggregation |
| Tracing | Jaeger (OpenTelemetry) | Distributed tracing |
| Alerting | Alertmanager + PagerDuty | Ops alerting |
| CDN / Edge Cache | Nginx + Cloudflare | Static asset delivery |
| VPN | WireGuard | Secure field connectivity |

---

## 6. Hardware & Infrastructure Requirements

### 6.1 Deployment Tiers

ATLAS supports three deployment profiles:

```
┌─────────────────┬──────────────────────┬───────────────────────┐
│  TIER           │  USE CASE            │  SCALE                │
├─────────────────┼──────────────────────┼───────────────────────┤
│  Cloud Core     │  Central HQ ops      │  Full capability      │
│  On-Prem DC     │  Secure facility     │  Full capability      │
│  Edge Node      │  Forward ops post    │  Reduced / offline    │
└─────────────────┴──────────────────────┴───────────────────────┘
```

---

### 6.2 Cloud Core (AWS / Azure / DigitalOcean)

#### Compute Nodes (Kubernetes Worker Nodes)

| Node Type | Spec | Count | Purpose |
|---|---|---|---|
| **General Purpose** | 16 vCPU, 64 GB RAM | 6 | API, stream processing, DB |
| **GPU — Inference** | 8x NVIDIA A100 80GB (or H100) | 2–4 | YOLO inference, LLM serving |
| **GPU — Training** | 8x NVIDIA A100 | 1–2 | Model fine-tuning |
| **Memory Optimized** | 32 vCPU, 256 GB RAM | 2 | Kafka, Elasticsearch |
| **Storage Optimized** | 16 vCPU, 64 GB RAM, 8TB NVMe | 4 | TimescaleDB, MinIO |

#### Storage

| Type | Size | Technology |
|---|---|---|
| Hot imagery storage | 100 TB | MinIO on NVMe SSD |
| Cold archive storage | 1 PB+ | MinIO on HDD / S3 Glacier |
| Database volumes | 10 TB | NVMe SSD (high IOPS) |
| Redis cache | 512 GB | In-memory (EBS/SSD backed) |

#### Network

| Component | Spec |
|---|---|
| Uplink | 100 Gbps backbone |
| External ingress | 10 Gbps |
| GPU interconnect | NVLink 4.0 / InfiniBand HDR (400 Gbps) |
| Latency (internal) | < 1ms pod-to-pod |

---

### 6.3 On-Premises Data Center

#### Server Rack Layout (2 x 42U Racks)

**Rack A — Compute**

| Slot | Hardware | Spec |
|---|---|---|
| 1–4 | GPU Inference Servers | Dell PowerEdge R750xa, 2x NVIDIA A100 80GB, 2x Xeon Gold 6348, 512 GB DDR4 |
| 5–8 | App / API Servers | Dell PowerEdge R650, 2x Xeon Silver 4316, 256 GB DDR4 |
| 9–12 | Stream Processing | Dell PowerEdge R650, 2x Xeon Gold 6330, 512 GB DDR4 |
| 13–16 | Kubernetes Control Plane | 3x Dell PowerEdge R450, 2x Xeon Silver 4310 |
| 17–20 | Edge Connectivity | Cisco ASR 1002-HX Routers |
| 21–24 | Load Balancers | 2x F5 BIG-IP i4800 |
| 25–28 | Network Switches | Cisco Nexus 9300 (100 GbE) |
| 29–32 | KVM / OOB Management | DELL iDRAC Enterprise |

**Rack B — Storage & Data**

| Slot | Hardware | Spec |
|---|---|---|
| 1–6 | NVMe Storage Arrays | Dell PowerStore 1200T (2x 100TB NVMe) |
| 7–12 | HDD Cold Storage | Dell PowerVault ME5024 (2x 500TB) |
| 13–16 | Tape Library (Archive) | Dell PowerVault TL2000 |
| 17–20 | Database Servers | 4x Dell PowerEdge R740xd, 2x Xeon Gold, 512 GB ECC RAM |
| 21–24 | Redis / Cache Cluster | 4x Supermicro 6029P-WTRT, 1.5 TB DRAM total |
| 25–28 | Backup Appliances | Veeam Backup on PowerEdge R450 |
| 29–32 | UPS + PDUs | APC Smart-UPS Ultra 20kW x2 |

#### Power Requirements
- Total power draw: **~80–100 kW** (peak)
- Recommended UPS capacity: **200 kVA** (with N+1 redundancy)
- Cooling: **Precision air cooling**, 30 tons CRAC units, hot-aisle containment

---

### 6.4 Edge Node (Forward Operations)

Designed to fit in a single ruggedized Pelican case or vehicle-mount rack.

| Component | Hardware | Spec |
|---|---|---|
| **Edge Server** | NVIDIA Jetson AGX Orin (x2) | 64-core ARM, 64 GB unified memory, 2048-core Ampere GPU |
| **Tactical Server** | Getac F110 rugged tablet / Intel NUC 13 Pro | 16-core, 64 GB RAM |
| **Storage** | Samsung T7 Shield SSD x4 | 4 TB total (ruggedized) |
| **Network** | Peplink MAX Transit Pro | Dual 5G/LTE, satellite (Starlink) fallback |
| **Switch** | Cisco IE-3400 Industrial Ethernet | -40°C to 75°C operating range |
| **Power** | Victron MultiPlus 24V/3000W | Solar + battery + shore power |
| **Display** | Xenarc 1000TSH (10" ruggedized) | 1000 nit, touch, IP65 |

**Edge Capabilities:**
- Runs: Object detection, local map tiles, basic COA, offline sync
- Battery life: 8–12 hours on battery pack
- Sync on reconnect: Delta sync via Kafka offset replay

---

### 6.5 End-User Devices

| Device | Spec | Role |
|---|---|---|
| **Operator Workstation** | Dell Precision 5860, 64 GB RAM, NVIDIA RTX 4090, 4K dual monitors | HQ operators |
| **Tactical Tablet** | Getac ZX10, 10.1" FHD, Android 12, IP67 | Field supervisors |
| **Ruggedized Laptop** | Panasonic Toughbook 55, i7, 32 GB, 14" FHD | Field analysts |
| **Phone (Field)** | Samsung Galaxy XCover7 (IP68) | Field operators |

---

### 6.6 Network Topology

```
[Internet / Satellite Uplink]
         │
    [WireGuard VPN Tunnel]
         │
    [Firewall Cluster — Palo Alto PA-5450]
         │
    [Core Switch — Cisco Nexus 9500]
         │ (10/100 GbE uplinks)
    ┌────┴────┐
    │         │
[Compute VLAN]  [Storage VLAN]  [Management VLAN]  [DMZ]
    │
[Kubernetes Pod Network — Calico CNI]
    │
[Service Mesh — Istio]
```

---

## 7. AI/ML Pipeline Design

### 7.1 Imagery Processing Pipeline

```
[Raw Frame Input]
  → Frame Extraction (ffmpeg / OpenCV)
  → Pre-processing: resize(640x640), normalize, pad
  → YOLOv10 Inference (Triton Server, TensorRT-optimized)
  → Post-processing: NMS, confidence filter (>0.45)
  → Detection schema: {frame_id, class, bbox, confidence, timestamp}
  → Kafka publish: topic/detections
  → DeepSORT tracker: assign/update track IDs
  → PostGIS write: track positions
  → WebSocket push: frontend
```

### 7.2 COA Generation Pipeline (RAG + LLM)

```
[Trigger: Operator requests COA or alert threshold crossed]
  → Context Builder:
      - Current tracked objects (PostGIS query)
      - Weather overlay (current + forecast)
      - Asset availability (logistics DB)
      - Historical patterns (Elasticsearch)
  → Embedding query → Weaviate (retrieve relevant doctrine/SOP)
  → Prompt assembly:
      SYSTEM: "You are a tactical decision support AI..."
      CONTEXT: [retrieved doctrine + current situation JSON]
      USER: "Generate 3 ranked COAs for: {situation_summary}"
  → LLaMA 3.1 70B inference (quantized Q4_K_M or FP16)
  → Parse structured JSON response
  → Store COA in PostgreSQL
  → Return to frontend with confidence + rationale
```

### 7.3 Anomaly Detection Pipeline

```
[TimescaleDB continuous aggregate: 1-min rolling stats]
  → Feature extraction: mean, std, rate-of-change
  → Isolation Forest scoring (online)
  → LSTM Autoencoder for sequence anomalies
  → Alert if: anomaly_score > threshold OR reconstruction_error > 3σ
  → Enrich alert with: sensor_id, location, probable_cause (LLM)
  → Route to: Alert Service → Operator queue
```

---

## 8. Data Architecture

### 8.1 Core Data Models

#### Track (Object)
```json
{
  "track_id": "uuid",
  "class": "vehicle|personnel|vessel|aircraft",
  "confidence": 0.92,
  "status": "active|lost|archived",
  "source_sensor_ids": ["sensor_001", "sensor_002"],
  "last_position": { "lat": 6.3350, "lon": 5.6037, "alt": 120 },
  "velocity_kmh": 45.2,
  "heading_deg": 270,
  "trajectory": [{ "lat": ..., "lon": ..., "timestamp": "..." }],
  "tags": ["priority", "vehicle-convoy"],
  "created_at": "2026-06-07T12:00:00Z",
  "updated_at": "2026-06-07T12:01:30Z"
}
```

#### Sensor
```json
{
  "sensor_id": "uuid",
  "type": "drone|satellite|iot|ais|adsb",
  "status": "online|offline|degraded",
  "location": { "lat": ..., "lon": ... },
  "stream_url": "rtsp://...",
  "metadata": { "model": "DJI Matrice 30T", "resolution": "4K" },
  "last_heartbeat": "2026-06-07T12:00:00Z"
}
```

#### COA (Course of Action)
```json
{
  "coa_id": "uuid",
  "situation_id": "uuid",
  "generated_at": "2026-06-07T12:00:00Z",
  "options": [
    {
      "rank": 1,
      "title": "Rapid interception via Route Alpha",
      "description": "...",
      "confidence": 0.87,
      "pros": ["..."],
      "cons": ["..."],
      "required_assets": ["unit_002", "unit_007"],
      "estimated_duration_min": 18
    }
  ],
  "selected_option": 1,
  "commander_notes": "..."
}
```

### 8.2 Data Retention Policy

| Data Type | Hot (TimescaleDB) | Warm (Elasticsearch) | Cold (MinIO) |
|---|---|---|---|
| Live tracks | 7 days | 90 days | 5 years |
| Raw imagery | 24 hours | — | 2 years |
| Sensor telemetry | 30 days | 1 year | 7 years |
| Audit logs | 90 days | 2 years | Indefinite |
| COAs / Reports | 1 year | — | Indefinite |

---

## 9. Security & Compliance

### 9.1 Zero-Trust Architecture

- Every service authenticates via mutual TLS (Istio)
- No implicit trust based on network location
- Short-lived JWTs (15 min expiry, refresh token rotation)
- Hardware Security Module (HSM) for key management

### 9.2 Data Classification

| Level | Description | Controls |
|---|---|---|
| **OPEN** | Weather, map tiles, public AIS | Unauthenticated read allowed |
| **INTERNAL** | Sensor configs, unit positions | Auth required, logged |
| **SENSITIVE** | Track data, COAs, imagery | RBAC + row-level security |
| **CLASSIFIED** | (Extension) | Air-gapped deployment only |

### 9.3 Compliance Targets

- **NIST SP 800-53** (for US government deployments)
- **ISO 27001** (international deployments)
- **GDPR** (EU civilian deployments)
- **SOC 2 Type II** (commercial SaaS offering)

---

## 10. Development Task Breakdown

### Phase 0 — Foundation (Weeks 1–4)

- [ ] **INFRA-001** — Provision Kubernetes cluster (k3s for dev, full K8s for prod)
- [ ] **INFRA-002** — Set up Kafka cluster (3 brokers, topic schema registry)
- [ ] **INFRA-003** — Deploy PostgreSQL + PostGIS + TimescaleDB
- [ ] **INFRA-004** — Deploy MinIO object storage
- [ ] **INFRA-005** — Set up Redis Cluster (6 nodes, 3 shards)
- [ ] **INFRA-006** — Deploy Kong API Gateway + Keycloak (OIDC)
- [ ] **INFRA-007** — CI/CD pipeline (GitHub Actions + ArgoCD)
- [ ] **INFRA-008** — Observability stack (Prometheus, Grafana, Loki, Jaeger)

### Phase 1 — Core Ingest & Map (Weeks 5–10)

- [ ] **SVC-001** — Ingest Service: MQTT broker + Kafka producer
- [ ] **SVC-002** — Ingest Service: RTSP drone stream → Kafka frames
- [ ] **SVC-003** — Ingest Service: REST polling adapters (weather, AIS, ADS-B)
- [ ] **SVC-004** — Geospatial Service: PostGIS CRUD + spatial query endpoints
- [ ] **SVC-005** — Geospatial Service: Map tile server (PMTiles + Mapbox Vector Tiles)
- [ ] **FE-001** — Web Dashboard: MapLibre GL base map + layer system
- [ ] **FE-002** — Web Dashboard: Real-time track rendering (WebSocket)
- [ ] **FE-003** — Web Dashboard: Layer toggle panel (weather, logistics, sensors)
- [ ] **FE-004** — Web Dashboard: Track detail sidebar + history replay

### Phase 2 — AI Vision Pipeline (Weeks 11–18)

- [ ] **ML-001** — Object Detection: YOLOv10 model setup + TensorRT export
- [ ] **ML-002** — Object Detection: Triton Inference Server deployment
- [ ] **ML-003** — Object Detection: Inference microservice (FastAPI)
- [ ] **ML-004** — Tracking Service: DeepSORT integration + track ID assignment
- [ ] **ML-005** — Tracking Service: Kalman filter trajectory prediction
- [ ] **ML-006** — Fusion Service: Multi-sensor track correlation engine
- [ ] **ML-007** — Pattern-of-life analysis: behavioral baseline per track
- [ ] **FE-005** — Web Dashboard: Detection overlay on video feed
- [ ] **FE-006** — Web Dashboard: Track timeline and pattern visualization

### Phase 3 — Decision Support (Weeks 19–26)

- [ ] **ML-008** — LLM Setup: Deploy LLaMA 3.1 70B (llama.cpp / vLLM)
- [ ] **ML-009** — Vector DB: Weaviate schema + doctrine ingestion pipeline
- [ ] **ML-010** — COA Service: Context builder + RAG retrieval
- [ ] **ML-011** — COA Service: LLM prompt chain + structured output parser
- [ ] **ML-012** — COA Service: Commander review + approval workflow
- [ ] **ML-013** — Alert Service: Rule engine (geofence, threshold)
- [ ] **ML-014** — Alert Service: Isolation Forest anomaly detector
- [ ] **FE-007** — Web Dashboard: COA panel + ranked options display
- [ ] **FE-008** — Web Dashboard: Alert triage queue + ack workflow

### Phase 4 — Logistics & Reporting (Weeks 27–32)

- [ ] **SVC-006** — Logistics Service: ERP CDC integration (Debezium)
- [ ] **SVC-007** — Logistics Service: Supply forecasting (Prophet model)
- [ ] **SVC-008** — Logistics Service: Route optimization API (pgRouting)
- [ ] **SVC-009** — Reporting Service: PDF/PPTX sitrep generation
- [ ] **FE-009** — Web Dashboard: Logistics overlay + supply indicators
- [ ] **FE-010** — Web Dashboard: Report builder UI

### Phase 5 — Mobile & Edge (Weeks 33–40)

- [ ] **MOB-001** — React Native tactical app: Map + track view
- [ ] **MOB-002** — React Native tactical app: COA display + commander actions
- [ ] **EDGE-001** — Flutter client: Offline SQLite data store
- [ ] **EDGE-002** — Flutter client: Background sync queue (Kafka offset-based)
- [ ] **EDGE-003** — Edge Server: Lightweight YOLOv10n deployment (Jetson)
- [ ] **EDGE-004** — Edge Server: Local tile server + offline map pack

### Phase 6 — Hardening & Launch (Weeks 41–48)

- [ ] **SEC-001** — Zero-trust Istio mTLS enforcement across all services
- [ ] **SEC-002** — HashiCorp Vault secrets rotation
- [ ] **SEC-003** — Penetration testing + red team exercise
- [ ] **SEC-004** — RBAC policy audit + row-level security enforcement
- [ ] **PERF-001** — Load test: 10,000 events/sec ingestion
- [ ] **PERF-002** — GPU inference benchmark: < 200ms per frame
- [ ] **PERF-003** — LLM benchmark: < 5s COA generation
- [ ] **DOC-001** — API documentation (OpenAPI 3.1)
- [ ] **DOC-002** — Operator training manual

---

## 11. Milestones & Roadmap

```
Q3 2026  ┌─────────────────────────────────────────┐
         │ PHASE 0 + 1: Infrastructure + Base Map  │
         └─────────────────────────────────────────┘
Q4 2026  ┌─────────────────────────────────────────┐
         │ PHASE 2: AI Vision Pipeline (MVP Alpha) │
         └─────────────────────────────────────────┘
Q1 2027  ┌─────────────────────────────────────────┐
         │ PHASE 3: Decision Support (Beta)        │
         └─────────────────────────────────────────┘
Q2 2027  ┌─────────────────────────────────────────┐
         │ PHASE 4: Logistics + Reporting          │
         └─────────────────────────────────────────┘
Q3 2027  ┌─────────────────────────────────────────┐
         │ PHASE 5: Mobile + Edge Clients          │
         └─────────────────────────────────────────┘
Q4 2027  ┌─────────────────────────────────────────┐
         │ PHASE 6: Hardening + Production Launch  │
         └─────────────────────────────────────────┘
```

### Team Composition (Recommended)

| Role | Count |
|---|---|
| Engineering Lead / Architect | 1 |
| Backend Engineers (Node.js / Python) | 4 |
| ML Engineers | 3 |
| Frontend Engineers (Next.js) | 2 |
| Mobile Engineer (React Native / Flutter) | 1 |
| DevOps / Platform Engineer | 2 |
| Security Engineer | 1 |
| Product Manager | 1 |
| UX Designer | 1 |
| **Total** | **16** |

---

## Appendix A — Estimated Build Cost (Self-Hosted)

| Category | Est. Monthly Cost (USD) |
|---|---|
| Cloud compute (K8s cluster, 20 nodes) | $12,000–18,000 |
| GPU instances (4x A100) | $14,000–20,000 |
| Storage (100TB NVMe + 1PB cold) | $3,000–5,000 |
| Network / egress | $1,000–2,000 |
| Third-party APIs (weather, AIS, ADS-B) | $500–1,500 |
| Monitoring / tooling SaaS | $500–1,000 |
| **Estimated Total (Cloud)** | **$31,000–47,500/mo** |
| On-prem CapEx (hardware purchase) | $800,000–1,200,000 one-time |

---

*Document generated June 2026. ATLAS is a civilian research reference implementation. All specifications are for educational and commercial disaster/logistics use. No classified information is included.*
