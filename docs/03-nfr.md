# VEKTOR — Non-Functional Requirements

[← Back to VEKTOR-PRD.md](../VEKTOR-PRD.md)

---

## 8. Non-Functional Requirements

### 8.1 Performance

| Metric | Requirement |
|---|---|
| Sensor ingest latency | < 50ms (arrival to normalized event) |
| CV inference throughput | ≥ 15 fps per stream via `onnxruntime-node` on GPU |
| Object detection latency | < 200ms per frame |
| Track fusion latency | < 500ms from detection to fused entity update |
| COA generation — cloud | < 10 seconds |
| COA generation — edge | < 30 seconds (`node-llama-cpp`, Mistral 7B Q4_K_M) |
| UI update latency | < 500ms from server event to rendered change |
| API response time (p99) | < 300ms for all read endpoints |
| Event ingestion throughput | ≥ 10,000 events/second sustained |
| Map performance | ≥ 60fps on WebGL map with 1,000+ active tracks |

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
| Vulnerability scanning | Automated SAST/DAST in CI; critical CVEs block deploy |

### 8.5 Usability

| Requirement | Specification |
|---|---|
| Device support | Renders correctly at 1024×768 minimum (tactical tablets) |
| Accessibility | WCAG 2.1 AA compliance |
| Offline UX | Offline state clearly communicated; stale-data timestamp shown |

### 8.6 Developer Experience

| Requirement | Specification |
|---|---|
| Core monorepo | Turborepo + pnpm workspaces (`vektor-backend`, `vektor-web`) |
| Language | TypeScript strict mode across all TS packages |
| Shared types | Generated from `vektor-proto`, re-exported via `packages/shared` |
| Test coverage | ≥ 80% unit coverage; integration tests for all service boundaries |
| Local dev | `pnpm dev` spins up the full platform stack via Docker Compose in < 2 minutes |
| Python isolation | Python services expose only gRPC interfaces; no TS service imports Python code, ever |

---

