# VEKTOR — Security, Compliance & Data Classification

[← Back to VEKTOR-PRD.md](../VEKTOR-PRD.md)

---

## 14. Security, Compliance & Data Classification

### 14.1 Threat Model

| Threat | Mitigation |
|---|---|
| Unauthorized data access | RBAC + row-level security + mTLS; all access logged |
| Sensor feed spoofing | Sensor auth tokens; feed integrity checksums via HMAC |
| LLM prompt injection via sensor metadata | Zod sanitization before prompt assembly; JSON-only context; output schema validation |
| Edge node physical compromise | Full-disk encryption (LUKS); remote wipe via WireGuard management channel |
| Insider threat | Principle of least privilege; anomalous access alerting via audit-svc |
| Supply chain (ONNX/GGUF model weights) | SHA-256 checksum verified at load time |
| Python enclave escape | `vektor-ml` runs in an isolated K8s namespace; no shared volumes with TS services; gRPC only (§9.3, §9.9) |
| Contract-boundary drift | `vektor-proto` requires dual-team review on every breaking change (§9.4) |

### 14.2 Python Enclave Security

The two `vektor-ml` services are treated as **untrusted third-party dependencies** from the perspective of the TS application layer:

- Run in an isolated K8s namespace (`vektor-ml-enclave`) with a network policy restricting all traffic except gRPC from `coa-svc` and `cv-train-svc` consumers
- No direct access to PostgreSQL, Redis, or MinIO — all storage access is proxied through TypeScript services
- Container images scanned via Trivy on every build; critical CVEs block deployment
- Resource quotas enforced (CPU, memory, GPU) to prevent runaway processes
- Repo-level enforcement: only ML-team members have write access to `vektor-ml` (§9.9)

### 14.3 Data Classification

| Level | Description | Controls |
|---|---|---|
| **OPEN** | Weather, map tiles, public AIS | Unauthenticated read allowed |
| **INTERNAL** | Sensor configs, unit positions | Auth required, logged |
| **SENSITIVE** | Track data, COAs, imagery | RBAC + row-level security |
| **RESTRICTED** | (Extension for classified/air-gapped deployments) | Air-gapped deployment only; no cloud tier |

### 14.4 Compliance Targets

| Standard | Applicability |
|---|---|
| NIST SP 800-53 | US government / defense-adjacent deployments |
| ISO 27001 | International commercial deployments |
| GDPR | EU civilian / disaster response deployments |
| SOC 2 Type II | Commercial SaaS offering |

---

