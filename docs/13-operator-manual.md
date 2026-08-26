# VEKTOR — Operator Training Manual & Runbooks

[← Back to VEKTOR-PRD.md](../VEKTOR-PRD.md)

---

## 20. Operator Training Manual

DOC-001 (roadmap §15, Phase 7). Written against the system as actually built through Phase 7 — every endpoint, role, and workflow named here exists in the codebase; this is not aspirational. Section numbers below are local to this doc (`§20.x`), not the PRD's original numbering.

### 20.1 Who this is for

This manual assumes you already know your role from [01-users-scope.md](01-users-scope.md) §5.1. Read your persona's section in §20.3 first; the runbooks in §21 are shared reference for whoever is on shift when an incident happens, regardless of role.

### 20.2 Signing in

Every human role authenticates via Keycloak SSO. A Systems Integrator (persona 7) is the one exception — it authenticates via a scoped API key, not a login, and doesn't use the operator dashboard at all (§20's audience is the dashboard's human users).

Your Keycloak role determines what you can see and do — not a setting you choose in the app. If a page or action you expect to have access to is missing, the fix is a role change requested through your SuperAdmin, not a workaround in the client.

### 20.3 Role Quick Reference

| Role | Owns | Cannot do |
|---|---|---|
| **Commander** | Approve/reject COAs (`POST /api/v1/coa/:coa_id/approve`\|`reject`), view target workbench ranking, create/delete geofence zones | Register sensors, upload models, query the audit log |
| **Analyst** | View/tag entities, view alerts, acknowledge/escalate/dismiss alerts, generate reports, request COA generation, flag hazard road edges | Approve a COA, create/delete geofence zones, manage inventory/routing |
| **Logistics Officer** | View inventory + depletion forecast, generate optimized routes | Approve a COA, tag entities, manage geofence zones |
| **Field Operator** | Same read surface as Analyst for offline-cached COP data (edge deployment); submit spot reports | Anything requiring live connectivity while offline; any Commander/SuperAdmin action |
| **SuperAdmin** | Register sensors, upload/hot-swap ONNX models, query the audit log, everything every other role can do | — |
| **Viewer** | Read-only: alerts list, sensor health | Any write action anywhere |

Full endpoint-by-role table: [07-data-api.md](07-data-api.md) §13.1.

### 20.4 The Common Operational Picture (COP)

The web dashboard (`apps/web`) renders every active entity as a MIL-STD-2525/APP-6-inspired symbol: shape = affiliation (rectangle/diamond/square/circle for friendly/hostile/neutral/unknown), color follows doctrine (friendly blue, hostile red, neutral green, unknown yellow), with a heading-rotated glyph distinguishing aircraft/vessel/EW-RF-emitter tracks. No-strike zones render as dashed red circles with a centroid label. The sensor health panel (bottom-left) shows live per-sensor status/latency/drop-rate.

Entities update live over the Socket.io gateway (`fusion-svc`); the map layer and sensor panel are always-mounted subscribers, so a page refresh is never required to see new activity — if the picture looks frozen, that's a connectivity or service-health problem (§21.4), not a UI setting to toggle.

### 20.5 Target Workbench (Commander/Analyst)

`GET /api/v1/target-workbench/ranking` ranks active entities by a 5-factor weighted threat score (SVC-012) — the algorithm is documented and auditable per REQ-4.1, not a black box. The ranking refreshes on every request (≤5s acceptance criteria), so a Commander triaging targets should expect the list order to shift as tracks update, corroborate, or go stale.

### 20.6 Requesting and Deciding a Course of Action

1. An Analyst requests generation (`POST /api/v1/coa/generate` against a target entity) — or `coa-svc` auto-triggers from an alert once that integration exists.
2. `coa-svc` returns up to 3 ranked options, each with a confidence score. Any option below the R-002 confidence threshold is rejected before it ever reaches this list — you will never see a COA option the system itself doesn't trust.
3. A Commander reviews (`GET /api/v1/coa/:situation_id`) and approves or rejects (`POST .../approve` or `.../reject`) with notes/reason. This is the human-in-the-loop mandate from R-002 — no COA executes without an explicit Commander decision.
4. The decision writes to the audit log synchronously, before the HTTP response returns (REQ-4.3) — by the time you see "approved" in the UI, the audit trail already has it.

### 20.7 Alerts

Alerts come from two sources: geofence entry/exit and EWMA/z-score anomaly detection (heading, speed). `GET /api/v1/alerts` (all roles) lists active alerts; `POST /api/v1/alerts/:id/actions` (Analyst+) records ACKNOWLEDGE, ESCALATE, or DISMISS, attributed to your verified identity — not something you can attribute to someone else. Multi-channel dispatch (email/webhook/SMS) fires independently of in-app acknowledgment; a channel failure is recorded, not silently dropped, and never blocks the other channels.

### 20.8 Logistics

`GET /api/v1/logistics/inventory` shows current stock plus a regression-based depletion forecast; a `LOW_STOCK` alert fires once per transition into low-stock (not on every 15-minute recompute, so your alert feed doesn't spam on a single sustained shortage). `POST /api/v1/logistics/route` avoids any road edge currently flagged as a hazard — flagging a new hazard (`POST /api/v1/logistics/road-edges/:id/block`, Analyst+) takes effect on the very next route request; there is no separate "recompute" step to wait on.

### 20.9 Reports

`POST /api/v1/reports` generates a PDF or PPTX sitrep synchronously (REQ-7.1's 30s bound). `GET /api/v1/export` produces raw CSV/GeoJSON for a time range and optional bounding box — useful for handing data to an external tool the Systems Integrator persona's API doesn't yet cover a workflow for.

---

## 21. Runbooks

Each runbook assumes you're looking at real symptoms, not hypotheticals — every one of these maps to a real failure mode found and fixed during this project's own build-out (see [10-risks-pitfalls.md](10-risks-pitfalls.md) and the [ADRs](adr/README.md) for the underlying engineering detail), not a generic incident-response template.

### 21.1 Sensor health / drop-rate breach (REQ-1.8)

**Symptom:** Sensor health panel flags a source above the 1% packet drop-rate threshold, or a source shows stale (no update within its expected interval).

1. Confirm it's the sensor, not the pipeline: check whether *other* sensors on the same domain (AIS/ADS-B/RTSP/MQTT) are also degraded. If yes, suspect the shared ingest path (`ingest-svc`, Kafka), not the individual sensor.
2. Check the sensor's own link/power status via its own management interface — VEKTOR reports what it receives, it doesn't diagnose the sensor itself.
3. If the sensor is confirmed offline, no further COP action is needed — its last-known track ages out naturally. If it's flapping (intermittent), log it for the SuperAdmin; a flapping source degrades EKF track quality (ADR-0003) more than a cleanly-offline one, since it corrupts an active filter's covariance rather than just leaving a gap.

### 21.2 A moving entity appears to "jump" or split into two tracks

**Symptom:** The same real-world contact appears as two disjointed entities, or a track's position jumps discontinuously between updates.

This is the exact failure mode ADR-0001 exists to prevent (ordering by `sensor_ts`, never arrival time). If you see it:

1. Check whether the affected sensor's clock sync (NTP/PTP) is healthy — SVC-004's daemon is host/infra config, and this is the most common real cause of this symptom.
2. If clock sync is confirmed healthy and this still reproduces, this is a `fusion-svc` bug report, not an operator workaround — do not manually merge or delete tracks to "fix" the display; that destroys audit trail.

### 21.3 A COA option's confidence looks wrong / COA generation returns nothing

**Symptom:** `POST /api/v1/coa/generate` returns fewer than the requested number of options, or none.

Every option below the R-002 confidence threshold is rejected before reaching the API response — a short or empty list is the confidence gate working as designed, not a bug. Check `coa-svc`'s logs for the pre-gate confidence scores if you need to confirm this rather than escalate it.

### 21.4 The COP looks frozen (no live updates)

1. Check browser devtools' network tab for the Socket.io connection state. A dropped/reconnecting socket means `apps/web` will show stale data until it reconnects — it does not silently show an error banner today (a known gap, not a hidden feature).
2. Confirm `fusion-svc` itself is up (`GET /healthz`) — it is both the fusion engine and the Socket.io gateway (SVC-007/FE-002 share the same process), so an outage here means both fused entities stop updating *and* the live picture stops, at the same time, from the same root cause.
3. If `fusion-svc` is healthy but the picture is still frozen, check whether your session's subscription mounted before or after the socket connected — a live-data hook gated behind a slow-to-ready component (e.g. waiting on the map's own `"load"` event) can silently miss the first post-connect emissions, since Socket.io does not replay past events to a late listener. This is a known class of bug (see [vektor-build-conventions] pattern notes); if reproducible, it's a frontend fix, not an operator action.

### 21.5 Model hot-swap (SuperAdmin)

**Symptom:** A new/updated detection model needs to go live without a service restart (REQ-2.6).

1. `POST /api/v1/models/upload` (SuperAdmin-only) with the `.onnx` file as multipart form data.
2. The upload itself *is* the validation step — a file that fails to load in `onnxruntime-node`, or that doesn't parse as a valid ONNX model, is rejected with a 400 and the currently-running model is left completely untouched. A rejected upload is not a partial or degraded state to clean up.
3. A successfully swapped model may have a different input size and output shape than the one it replaced (different `imgsz`, different class count) — this is expected and handled; do not assume a swap failed just because inference output shapes changed.
4. If `REQUIRE_GPU=true` was set for this deployment and the GPU execution provider isn't actually available, the service fails fast at startup rather than silently falling back to a much slower CPU path (Pitfall 5) — a "won't start" symptom here means check the GPU/CUDA toolkit match, not the model file.

### 21.6 Edge node reconnects after an offline period

**Symptom:** A Tactical Edge Node (§11.4) that was disconnected (per §11.1, rated for up to 72h with zero connectivity) comes back online.

Delta sync replays from the last acknowledged Kafka offset on reconnect — it is not a full state replay. Locally-queued offline writes (IndexedDB → BullMQ intake queue) drain automatically once connectivity returns; there is no manual "sync now" action required or available. If sync appears stuck well past what the queue size would suggest, escalate as an `edge-sync-svc` incident rather than restarting the node — a restart mid-drain risks losing whatever hadn't yet been persisted from the intake queue.

### 21.7 Audit log review (SuperAdmin)

`GET /api/v1/audit` (SuperAdmin-only) supports filtering by actor, action, and time range. The log is append-only, enforced two ways: by convention (no service anywhere implements an update/delete query against it) and, since Phase 7, by Postgres row-level security as a DB-layer backstop (ADR-0011) — an entry you see here cannot have been altered after the fact by any application-layer bug, only by a superuser database connection, which is itself outside the RBAC model this manual covers and should be treated as a break-glass action requiring separate justification.

---

See [ADRs](adr/README.md) for the engineering reasoning behind decisions referenced above, and [10-risks-pitfalls.md](10-risks-pitfalls.md) for the underlying incident detail.
