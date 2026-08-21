# VEKTOR — Stakeholders, User Types & Scope

[← Back to VEKTOR-PRD.md](../VEKTOR-PRD.md)

---

## 4. Stakeholders & Team

| Role | Responsibility |
|---|---|
| Product Owner | Requirements sign-off, roadmap prioritization |
| Engineering Lead / Architect | Architecture decisions, cross-repo delivery |
| ML / CV Lead | Model selection, training pipeline, inference optimization |
| Security Lead | Threat modeling, compliance, penetration testing, cross-repo access policy |
| DevOps / Platform Lead | Infrastructure, CI/CD, edge deployments, the `vektor-infra` repo |
| End Users | Operations commanders, analysts, logistics officers, field operators — requirements input, UAT |
| Executive Sponsor | Budget, escalation, strategic alignment |

Recommended team composition and cost model in §18.

---

## 5. User Types

Earlier drafts carried two disconnected lists — five UX personas and six RBAC roles — that didn't map 1:1 (two RBAC roles had no persona; one persona had no RBAC role). §5.1 is the single source of truth: every human user type ties to exactly one RBAC role, and the one non-human consumer is called out as authenticating differently on purpose, not by omission.

### 5.1 Unified User-Type Model

| # | User Type | RBAC Role | Authenticates via | Primary Goal |
|---|---|---|---|---|
| 1 | Operations Commander | `Commander` | Keycloak SSO (human) | Own the COP; approve/reject COAs |
| 2 | Intelligence / Situational Analyst | `Analyst` | Keycloak SSO (human) | Classify, annotate, and investigate entities |
| 3 | Logistics Officer | `Logistics Officer` | Keycloak SSO (human) | Inventory visibility, routing, forecasting |
| 4 | Field Operator (Edge / Dismounted) | `Field Operator` | Keycloak SSO (human), offline-cached token | Offline COP access; spot reports |
| 5 | Platform Administrator | `SuperAdmin` | Keycloak SSO (human) | Sensor registration, RBAC policy, model hot-swap, audit access |
| 6 | Read-Only Observer | `Viewer` | Keycloak SSO (human) or scoped share link | Briefing-only, read-only COP access |
| 7 | Systems Integrator | *— not an RBAC role —* | Scoped OAuth2 client-credentials API key | Third-party integration via the public API (§Epic 9) |

**Why the Systems Integrator has no RBAC role:** every RBAC role in the table above assumes a human sitting behind a Keycloak login with a role that can change (promotion, reassignment, offboarding). A partner integration is a service account: it should authenticate against a scoped API key with its own rate limits and revocation path (REQ-8.5), independent of the human role matrix — collapsing the two would mean either over-privileging integrators with a human role, or under-specifying their access entirely. The two are deliberately separate systems that happen to sit in the same table.

### 5.2 Persona Detail

### Persona 1 — Operations Commander (`Commander`)

**Profile:** Senior leader at a command center. Needs an immediate, holistic situational picture. Makes high-stakes, time-sensitive decisions.

**Goals:** See everything on one screen. Get ranked recommended actions in plain language. Approve or reject with one tap.

**Frustrations:** Stale data from disconnected tools. Drowning in raw video feeds with no synthesis.

**Key Use Cases:**
- `UC-1.1` View live Common Operational Picture (COP) across all active sensors
- `UC-1.2` Review and approve/modify/reject an AI-generated Course of Action (COA)
- `UC-1.3` Declare a No-Strike/No-Interdiction Zone around friendly or protected assets

---

### Persona 2 — Intelligence / Situational Analyst (`Analyst`)

**Profile:** Mid-level analyst reviewing imagery, tracks, and RF data. Produces intelligence/incident reports. Works long shifts at HQ or a forward post.

**Goals:** Quickly identify, classify, and annotate objects of interest. Build pattern-of-life profiles on tracked entities.

**Frustrations:** Manual frame-by-frame video review. No persistent object IDs when targets go behind cover.

**Key Use Cases:**
- `UC-2.1` Review auto-generated detections and correct misclassifications
- `UC-2.2` Query historical track data for a specific entity
- `UC-2.3` Export annotated imagery and entity reports

---

### Persona 3 — Logistics Officer (`Logistics Officer`)

**Profile:** Manages supply chain, asset inventory, and route planning. Needs to know what assets are available and where.

**Goals:** Real-time inventory visibility. Predictive resupply before shortfalls. Optimized routing that avoids current threat/hazard zones.

**Frustrations:** ERPs disconnected from the operational picture. No forecasting capability. Route plans not updated with current positions.

**Key Use Cases:**
- `UC-3.1` View asset inventory levels with depletion forecasts
- `UC-3.2` Generate an optimized supply route avoiding flagged threat/hazard zones
- `UC-3.3` Receive predictive low-stock alerts before depletion

---

### Persona 4 — Field Operator, Edge / Dismounted (`Field Operator`)

**Profile:** On-the-ground operator with a ruggedized tablet. Limited or no connectivity. Needs a lightweight, fast interface.

**Goals:** Access the current picture offline. Sync data instantly on reconnect. Simple UI that works with gloves.

**Frustrations:** Full dashboard too heavy for field hardware. Loses all data visibility when off-network.

**Key Use Cases:**
- `UC-4.1` View last-known track positions and COA offline
- `UC-4.2` Submit a spot report that queues for sync on reconnect
- `UC-4.3` Receive push alerts when connectivity restores

---

### Persona 5 — Platform Administrator (`SuperAdmin`)

**Profile:** Owns the operational health of the deployment itself — sensor onboarding, model releases, role assignment, incident audit. Rarely touches the tactical workflow; lives in the admin surfaces.

**Goals:** Onboard new sensors quickly and safely. Ship a hot-swapped detection model with confidence. Answer "who did what, when" without digging through raw logs.

**Frustrations:** No single audit view — has to correlate Keycloak, Kafka, and app logs by hand. Model uploads with no validation step risk taking down inference silently.

**Key Use Cases:**
- `UC-5.1` Register a new sensor source and verify it appears healthy on the ingestion dashboard
- `UC-5.2` Upload and hot-swap a custom ONNX detection model (REQ-2.6)
- `UC-5.3` Query the immutable audit log by user, time range, and action type

---

### Persona 6 — Read-Only Observer (`Viewer`)

**Profile:** Needs situational awareness without the authority (or need) to act — an auditor, a trainee, an executive sitting in on a briefing, a partner agency watching a shared AOI.

**Goals:** See the current COP and relevant history without being able to change anything. Trust that sensitive fields are actually hidden, not just visually de-emphasized.

**Frustrations:** Tools that either lock them out entirely or, worse, silently expose data they weren't cleared to see.

**Key Use Cases:**
- `UC-6.1` View the live COP for an authorized area of interest, read-only
- `UC-6.2` Review historical entity data without access to analyst notes or COA rationale
- `UC-6.3` Join a briefing via a scoped, time-limited share link with no account required

---

### Persona 7 — Systems Integrator / Third-Party Developer (service account, not RBAC)

**Profile:** Engineer at a partner org integrating VEKTOR's public API into an adjacent tool (fleet management, weather service, custom sensor). Never logs into the operator dashboard.

**Goals:** Stable, versioned, well-documented API and event contracts. Predictable breaking-change policy. Clear rate limits and key-scoping, independent of any human's role.

**Frustrations:** Undocumented internal APIs; breaking changes with no notice; access models designed only for humans.

**Key Use Cases:**
- `UC-7.1` Register a new sensor source via the public API using a scoped API key
- `UC-7.2` Subscribe to entity/alert WebSocket events for a bounded area
- `UC-7.3` Pull historical entity data via REST for offline analytics

---

## 6. Scope & Boundaries

### 6.1 In Scope — v1.0

- Multi-source sensor ingestion (FMV, SAR, satellite, EW, AIS, ADS-B, IoT/MQTT, weather)
- Real-time object detection and classification via ONNX model served by `onnxruntime-node`
- Multi-object tracking with persistent IDs (DeepSORT logic re-implemented in TypeScript)
- Multi-sensor data fusion via Kafka + Redis Streams stateful processing
- Blue-force/friendly-asset tracking with automatic no-strike zone enforcement
- COA generation via `node-llama-cpp` (edge) and gRPC call to a Python LLM-serving enclave (cloud)
- Target workbench with human-in-the-loop approval
- Logistics: supply forecasting, route optimization, ERP CDC sync
- Reporting: PDF/PPTX sitrep generation, raw data export (CSV/GeoJSON)
- Role-based access control (RBAC) enforced in the TypeScript API layer
- Web dashboard (Next.js / MapLibre / CesiumJS)
- Edge deployment mode (Jetson AGX Orin, k3s, 72h offline)
- Public REST/WebSocket API for third-party integration

### 6.2 Out of Scope — v1.0 (Candidate for v2.0)

- Autonomous strike/interdiction execution (**permanently** out of scope — human approval always required)
- SIGINT / signals-intelligence deep processing pipeline
- Voice command interface
- Federated/allied data exchange (MIL-STD-2525 and similar standards)
- Simulation / "what-if" wargaming mode
- Native mobile app (React Native / Flutter) — v1 ships PWA only; native app is a v2 candidate if field-device requirements demand it
- AI model marketplace (plug-in custom detection models per deployment)

---

