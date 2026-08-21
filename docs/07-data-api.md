# VEKTOR — Data Architecture & API Contracts

[← Back to VEKTOR-PRD.md](../VEKTOR-PRD.md)

---

## 12. Data Architecture & Core Models

### 12.1 Shared Type Package

All data schemas are defined once in `vektor-proto` (`src/schemas/`) and re-exported by `packages/shared` — no schema duplication. `packages/shared` never redeclares a shape; it only re-exports what `@vektor/proto` already validated at the contract-repo level (see `vektor-proto/README.md` and §9.4).

```typescript
// packages/shared/src/index.ts
export * from '@vektor/proto'
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

`DetectionEvent` and `SensorHealth` are defined once in `vektor-proto/src/schemas/detection-event.ts` (see Pitfall 1, §17, on why `sensor_ts` — not `kafka_ts` — is the only valid ordering field). `packages/kafka` imports them rather than redeclaring:

```typescript
// packages/kafka/src/schemas/detection.ts
export { DetectionEvent, SensorHealth } from '@vektor/proto'
```

**Topic naming convention:** `{env}.vektor.{domain}.{action}`
Examples: `prod.vektor.video.frame`, `prod.vektor.entity.updated`, `prod.vektor.alert.fired`

### 12.4 Data Retention Policy

| Data Type | Hot (TimescaleDB) | Warm (Elasticsearch) | Cold (MinIO) |
|---|---|---|---|
| Live entity tracks | 7 days | 90 days | 5 years |
| Raw FMV clips | 24 hours | — | 1 year |
| Imagery (satellite/SAR) | 7 days | — | 2 years |
| Sensor telemetry | 30 days | 1 year | 7 years |
| Audit logs | 90 days | 2 years | Indefinite |
| ONNX/GGUF model weights | Current only (MinIO) | — | All versions |
| COAs & Reports | 1 year | — | Indefinite |

---

## 13. API Contracts

### 13.1 REST Endpoints (Fastify + TypeScript)

All endpoints are typed end-to-end, generated from `vektor-proto` and referenced in both Fastify route handlers and frontend API clients.

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
| `GET` | `/api/v1/logistics/inventory` | Asset inventory with depletion forecast | Logistics+ |
| `POST` | `/api/v1/logistics/route` | Generate optimized route avoiding flagged zones | Logistics+ |
| `GET` | `/api/v1/reports/:id` | Retrieve a generated sitrep (PDF/PPTX) | Analyst+ |

### 13.2 WebSocket Events (Socket.io)

The event-name → payload-schema maps are defined once in `vektor-proto/src/schemas/socket-events.ts` (`ServerToClientEvents`, `ClientToServerEvents`, both Zod). `apps/web`'s Socket.io client/server typings are derived from those maps, not hand-typed:

```typescript
// apps/web/src/lib/socket-types.ts
import { z } from 'zod'
import { ServerToClientEvents as S2CSchemas, ClientToServerEvents as C2SSchemas } from '@vektor/proto'

export type ServerToClientEvents = {
  [K in keyof typeof S2CSchemas]: (payload: z.infer<(typeof S2CSchemas)[K]>) => void
}
export type ClientToServerEvents = {
  [K in keyof typeof C2SSchemas]: (payload: z.infer<(typeof C2SSchemas)[K]>) => void
}
```

### 13.3 gRPC Interface (TS ↔ Python Boundary, defined in `vektor-proto`)

```protobuf
// vektor-proto/llm.proto — consumed by coa-svc (vektor-platform) and llm-cloud-svc (vektor-ml)
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

