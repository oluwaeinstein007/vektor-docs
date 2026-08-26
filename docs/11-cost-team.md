# VEKTOR — Cost Model & Team Composition

[← Back to VEKTOR-PRD.md](../VEKTOR-PRD.md)

---

## 18. Cost Model & Team Composition

### 18.1 Recommended Team

| Role | Count | Primary Repo(s) |
|---|---|---|
| Engineering Lead / Architect | 1 | all |
| Backend Engineers (TypeScript) | 5 | `vektor-backend` |
| ML Engineers (Python) | 3 | `vektor-ml` |
| Frontend Engineers (Next.js) | 2 | `vektor-web` |
| DevOps / Platform Engineers | 2 | `vektor-infra`, `vektor-edge` |
| Security Engineer | 1 | `vektor-infra`, cross-repo audit |
| Product Manager | 1 | — |
| UX Designer | 1 | — |
| **Total** | **16** | |

### 18.2 Estimated Monthly Build Cost (Self-Hosted Cloud Core)

| Category | Est. Monthly Cost (USD) |
|---|---|
| Cloud compute (K8s cluster, ~20 nodes) | $12,000–18,000 |
| GPU instances (4x H100-class) | $18,000–26,000 |
| Storage (100 TB NVMe + 1 PB cold) | $3,000–5,000 |
| Network / egress | $1,000–2,000 |
| Third-party data APIs (weather, AIS, ADS-B) | $500–1,500 |
| Monitoring / tooling SaaS | $500–1,000 |
| **Estimated Total (Cloud)** | **$35,000–53,500/mo** |
| On-prem CapEx (hardware purchase, one-time) | $850,000–1,250,000 |

---

