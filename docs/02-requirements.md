# VEKTOR — Functional Requirements (Epics)

[← Back to VEKTOR-PRD.md](../VEKTOR-PRD.md)

---

## 7. Functional Requirements (Epics)

> **Priority Scale:** P0 = Must ship (blocking), P1 = High value, P2 = Important, P3 = Nice to have

---

### Epic 1 — Multimodal Sensor Ingest & Normalization

**Goal:** Accept raw data from all sensor types and normalize to a canonical, time-aligned, coordinate-normalized schema before any downstream processing.

| ID | Requirement | Priority | Acceptance Criteria |
|---|---|---|---|
| REQ-1.1 | Continuously ingest FMV from drone feeds via RTSP and SRT protocols | P0 | ≥ 30fps ingest with < 50ms ingest-to-normalize latency |
| REQ-1.2 | Ingest SAR imagery and commercial satellite GeoTIFF files | P0 | GeoTIFF processed within 5s of upload; SAR imagery georeferenced correctly |
| REQ-1.3 | Normalize all spatial data to WGS 84 / MGRS with sub-meter precision | P0 | Spatial alignment error < 1m across all source types |
| REQ-1.4 | Synchronize all sensor streams using NTP/PTP clock alignment | P0 | Timestamp skew across sensors < 5ms after sync |
| REQ-1.5 | Support ingestion of AIS (maritime), ADS-B (aircraft), and MQTT IoT telemetry | P1 | New objects appear on COP within 10s of message receipt |
| REQ-1.6 | Ingest weather data (NOAA/OpenWeatherMap-class APIs) as a COP overlay | P1 | Weather layer refreshes ≤ every 10 minutes |
| REQ-1.7 | Sync logistics/ERP state via CDC (Debezium-class change-data-capture) | P1 | ERP delta reflected in VEKTOR within 60s |
| REQ-1.8 | Expose an ingestion health dashboard: per-source status, latency, drop rate | P1 | Real-time status; alert fires if any source drops > 1% of packets |

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

### Epic 2 — Computer Vision & Automated Object Tracking

**Goal:** Run continuous AI inference on all incoming video/imagery to detect, classify, and persistently track objects of interest without manual intervention.

| ID | Requirement | Priority | Acceptance Criteria |
|---|---|---|---|
| REQ-2.1 | Run real-time CV inference on all active FMV streams simultaneously | P0 | ≥ 15fps inference per stream on standard GPU node |
| REQ-2.2 | Detect and classify target profiles with confidence ≥ 85% | P0 | mAP ≥ 0.85 on held-out validation set; false positive rate < 5% |
| REQ-2.3 | Render bounding boxes with class label and confidence score on UI | P0 | Renders within 500ms of detection event |
| REQ-2.4 | Maintain persistent track IDs across frames, including occlusion up to 10s | P0 | ID retention rate ≥ 90% in simulated occlusion scenarios |
| REQ-2.5 | Estimate velocity and heading via optical flow / Kalman filtering | P1 | Velocity estimates within 10% of ground truth |
| REQ-2.6 | Support custom ONNX model upload for domain-specific detection classes | P2 | Upload → validate → hot-load in `onnxruntime-node` within 5 minutes, no service restart |
| REQ-2.7 | Pattern-of-life analysis: behavioral baseline per persistent track | P2 | Baseline computed after 24h of observations; deviation flagged for analyst review |

---

### Epic 3 — Data Fusion & Unified Ontology Layer

**Goal:** Correlate detections from all sensors into single, deduplicated entity records using a canonical ontology. Eliminate phantom tracks and cross-domain duplication.

| ID | Requirement | Priority | Acceptance Criteria |
|---|---|---|---|
| REQ-3.1 | Cross-reference CV tracks with EW/RF emissions, AIS, ADS-B, and satellite telemetry to build unified entity state | P0 | ≥ 95% dedup rate on synthetic multi-sensor test scenarios |
| REQ-3.2 | Map all entity classifications strictly to the central domain ontology schema | P0 | 100% of output objects conform; schema violations logged and quarantined |
| REQ-3.3 | Enforce blue-force/friendly-asset tracking with automatic no-strike zone application | P0 | Zones render within 1s of position update; system rejects any COA targeting a protected asset |
| REQ-3.4 | Maintain full entity state history — position, velocity, classification changes, sensor source | P1 | History queryable up to 30 days; time-scrub replay available in UI |
| REQ-3.5 | Detect and flag conflicting classifications across sensors for analyst review | P1 | Conflict detected within 5s; queued to analyst review panel |
| REQ-3.6 | Entity similarity search via vector embeddings | P2 | Top-10 results in < 500ms on Qdrant index |

**Canonical Entity Schema:**

```typescript
// packages/shared/src/types/entity.ts  (published from vektor-proto → consumed here)
export interface Entity {
  entity_id: string;            // UUID v4
  classification: string;       // e.g. "GroundVehicle.Tracked"
  confidence: number;           // 0.0 – 1.0
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
  first_detected: string;       // ISO 8601
  last_updated: string;         // ISO 8601
}
```

---

### Epic 4 — Tactical AI Recommendations & Target Workbench

**Goal:** Provide operators with a prioritized target queue and AI-generated response recommendations that they can approve, modify, or reject — human-in-the-loop at all times.

| ID | Requirement | Priority | Acceptance Criteria |
|---|---|---|---|
| REQ-4.1 | Rank active entities in Target Workbench by threat/priority score | P0 | Ranking refreshes ≤ every 5s; algorithm documented and auditable |
| REQ-4.2 | Generate ≥ 3 ranked COA options via self-hosted LLM, matched against asset inventory | P0 | COA in < 10s (cloud) / < 30s (edge); structured JSON validated against schema |
| REQ-4.3 | COA interface: approve/modify/reject with one tap; all decisions logged | P0 | Approval publishes target package within 2s; full audit trail preserved |
| REQ-4.4 | COA generation factors in weather, terrain, assets, no-strike zones, friendly positions | P1 | Test suite validates 100% no-strike-zone compliance |
| REQ-4.5 | Show COA reasoning transparency — context factors listed per recommendation | P1 | Operator can expand a rationale panel; context sources listed |
| REQ-4.6 | Escalate unactioned targets beyond configurable time thresholds | P2 | Escalation alert fires at T+threshold; webhook dispatched |

---

### Epic 5 — Alerting & Anomaly Detection

**Goal:** Proactively surface threats and anomalies before operators notice them manually.

| ID | Requirement | Priority | Acceptance Criteria |
|---|---|---|---|
| REQ-5.1 | Trigger geofence alerts on zone entry/exit | P0 | Alert fires within 3s of boundary crossing |
| REQ-5.2 | Detect behavioral anomalies (speed/heading/formation deviation) via statistical time-series analysis | P1 | False positive rate < 10% on validation dataset |
| REQ-5.3 | Route alerts via in-app queue, email, SMS, and configurable webhook | P1 | Delivered to all channels within 10s of trigger |
| REQ-5.4 | Alert triage queue: acknowledge, escalate, dismiss; all actions logged | P1 | All alert actions logged with operator ID and timestamp |

---

### Epic 6 — Logistics Intelligence

**Goal:** Give the Logistics Officer persona a predictive, real-time view of assets and supply instead of a disconnected ERP.

| ID | Requirement | Priority | Acceptance Criteria |
|---|---|---|---|
| REQ-6.1 | Real-time asset inventory with depletion-rate forecasting | P1 | Forecast recomputed every 15 min; visible on logistics overlay |
| REQ-6.2 | Route optimization avoiding flagged threat/hazard zones | P1 | Route recomputed within 5s of a new hazard flag along an active route |
| REQ-6.3 | Predictive low-stock alerts before depletion | P2 | Alert fires at configurable lead-time threshold before projected stock-out |

---

### Epic 7 — Reporting & Export

| ID | Requirement | Priority | Acceptance Criteria |
|---|---|---|---|
| REQ-7.1 | Auto-generate PDF/PPTX situation reports | P1 | Report generated within 30s of request |
| REQ-7.2 | Scheduled report dispatch | P2 | Reports delivered on configured cron schedule |
| REQ-7.3 | Raw data export (CSV, GeoJSON) | P2 | Export completes within 60s for a 30-day, single-AOI query |

---

### Epic 8 — Access Control, Audit & Compliance

**Goal:** Enforce strict data access governance and maintain a complete, tamper-proof audit trail.

| ID | Requirement | Priority | Acceptance Criteria |
|---|---|---|---|
| REQ-8.1 | RBAC enforces the six human roles in the unified user-type model (§5.1): SuperAdmin, Commander, Analyst, Logistics Officer, Field Operator, Viewer | P0 | Role matrix enforced at API level; unauthorized access returns 403 within 50ms |
| REQ-8.2 | Row-level and column-level data security based on user role | P0 | Sensitive fields hidden from Viewer/Field Operator in all API responses |
| REQ-8.3 | Immutable audit log for every access, modification, and approval action | P0 | Written to append-only store; no update/delete permitted |
| REQ-8.4 | Short-lived JWTs (15min), refresh token rotation, forced re-auth on role change | P0 | Expired token returns 401; no access after forced logout |
| REQ-8.5 | Systems Integrators (§5.1, Persona 7) authenticate via scoped OAuth2 client-credentials API keys, never via a human RBAC role | P1 | Key scopes enforced independently of the RBAC matrix; revoked key rejected within 60s |

---

### Epic 9 — Public API & Third-Party Integration

**Goal:** Support Persona 7 (Systems Integrator, §5.1) with a stable, versioned, documented API surface.

| ID | Requirement | Priority | Acceptance Criteria |
|---|---|---|---|
| REQ-9.1 | Versioned public REST API (`/api/v1/...`) with OpenAPI 3.1 spec | P1 | 100% of public endpoints documented and schema-validated |
| REQ-9.2 | WebSocket subscription API scoped to a bounding box | P1 | Client receives only events within its subscribed AOI |
| REQ-9.3 | Deprecation policy: 90-day notice before any breaking change to a public contract | P2 | Enforced via `vektor-proto` versioning (§9.4) |

### 7.1 Phase 2 / Advanced Feature Candidates (Post-v1.0)

Captured here so they aren't lost, but explicitly **not** committed for v1.0:

| Feature | Description |
|---|---|
| Multi-domain fusion | Correlate air, land, sea, space, cyber tracks in one ontology |
| Simulation / wargaming mode | Run "what-if" scenarios without touching live data |
| AI model marketplace | Plug in custom detection models per deployment |
| Federated/allied data sharing | Standards-based data exchange (e.g., MIL-STD-2525-class protocols) |
| Voice command interface | LLM-powered natural-language operator commands |
| Automated SIGINT analysis | Deep signals-intelligence processing pipeline |
| Native mobile app | React Native (tablet) / dedicated field app, if PWA proves insufficient |

---

