# VEKTOR Documentation Map

[← Back to VEKTOR-PRD.md](../VEKTOR-PRD.md) for the naming decision and executive summary.

Each file below is a self-contained slice of the original master PRD, split so any one concern can be read, linked to, or reviewed independently. Section numbers (`§9`, `REQ-8.1`, etc.) referenced inside a file point at the *original* numbering scheme below — they are not clickable across files, just a pointer to which doc to open next.

| # | Section | File | What's in it |
|---|---|---|---|
| 2–3 | Problem Statement · Goals & Success Metrics | [00-vision.md](00-vision.md) | Pain points, desired future state, primary goals/metrics, anti-goals |
| 4–6 | Stakeholders & Team · User Types · Scope & Boundaries | [01-users-scope.md](01-users-scope.md) | The unified 7-user-type model (§5.1), all persona detail, v1.0 in/out of scope |
| 7 | Functional Requirements (Epics) | [02-requirements.md](02-requirements.md) | Epics 1–9, every `REQ-x.x`, Phase 2 candidates |
| 8 | Non-Functional Requirements | [03-nfr.md](03-nfr.md) | Performance, availability, scalability, security, usability, DX targets |
| 9 | System Design & Repository Architecture | [04-system-design.md](04-system-design.md) | The 7-repo polyrepo layout, `vektor-proto` contract boundary, CI/CD topology, access control |
| 10 | Tech Stack | [05-tech-stack.md](05-tech-stack.md) | Frontend, backend, AI/ML, storage, infra — by repo |
| 11 | Hardware Architecture & Deployment | [06-hardware.md](06-hardware.md) | Cloud core, on-prem rack layout, edge node BOM, end-user devices |
| 12–13 | Data Architecture · API Contracts | [07-data-api.md](07-data-api.md) | Entity schema, Drizzle/Kafka schemas, REST/WebSocket/gRPC contracts |
| 14 | Security, Compliance & Data Classification | [08-security-compliance.md](08-security-compliance.md) | Threat model, Python enclave security, data classification, compliance targets |
| 15 | Development Roadmap | [09-roadmap.md](09-roadmap.md) | Phase 0–7, task IDs, owning repo per task |
| 16–17 | Risk Register · Known Engineering Pitfalls | [10-risks-pitfalls.md](10-risks-pitfalls.md) | R-001–R-011, the 6 documented pitfalls with mitigations |
| 18 | Cost Model & Team Composition | [11-cost-team.md](11-cost-team.md) | Recommended headcount, monthly cloud cost, on-prem CapEx |
| 19 | Glossary | [12-glossary.md](12-glossary.md) | All acronyms and VEKTOR-specific terms |
| 20–21 | Operator Training Manual · Runbooks | [13-operator-manual.md](13-operator-manual.md) | DOC-001: per-role quick reference, common workflows, incident runbooks |
| 22–25 | SEC-004 Security Audit | [14-sec004-security-audit.md](14-sec004-security-audit.md) | Dependency vulnerability audit, pentest engagement plan, SEC-003 self-audit results |
| 26–27 | Phase 7 Performance Benchmark Results | [15-perf-results.md](15-perf-results.md) | PERF-001's real measured Kafka throughput; PERF-002/003/004's hardware-availability gap |
| — | Architecture Decision Records | [adr/](adr/README.md) | DOC-002/R-011: why each major implementation-time decision was made, not just what the pitfall was |
| — | **Current System Status** | [16-system-status.md](16-system-status.md) | **Start here.** A dated, living snapshot of what's actually built/verified today, beyond the original Phase 0–7 roadmap — how to bring up a local dev stack, known gaps, multi-session coordination notes |
| — | **Next Work: Tasks & Features** | [17-next-work.md](17-next-work.md) | The post-roadmap backlog: `NW-xxx` tasks tiered by what unblocks what, from "nothing ships until this is done" to the features that open new markets |
| — | **Use Cases & Target Markets** | [18-use-cases.md](18-use-cases.md) | What VEKTOR can verifiably do today, eight candidate verticals ranked by fit, and which wedge to lead with |
| — | **Commercialization & Path to Revenue** | [19-commercialization.md](19-commercialization.md) | Licensing decision, pricing unit and packaging, revenue lines beyond the licence, and a staged path from zero customers to repeatable |

## Repo-to-doc quick reference

If you're working in a specific VEKTOR repo and want just the relevant spec:

| Repo | Start with |
|---|---|
| `vektor-backend` | [02-requirements.md](02-requirements.md), [05-tech-stack.md](05-tech-stack.md), [07-data-api.md](07-data-api.md) |
| `vektor-web` | [02-requirements.md](02-requirements.md) (Epic covering FE-xxx tasks), [09-roadmap.md](09-roadmap.md) Phase 1 |
| `vektor-ml` | [04-system-design.md](04-system-design.md) §9.9, [05-tech-stack.md](05-tech-stack.md) §10.3, [08-security-compliance.md](08-security-compliance.md) §14.2 |
| `vektor-proto` | [04-system-design.md](04-system-design.md) §9.4, [07-data-api.md](07-data-api.md) §13.3 |
| `vektor-edge` | [06-hardware.md](06-hardware.md) §11.4, [09-roadmap.md](09-roadmap.md) Phase 6, [10-risks-pitfalls.md](10-risks-pitfalls.md) Pitfall 2 |
| `vektor-infra` | [04-system-design.md](04-system-design.md) §9.8–9.9, [06-hardware.md](06-hardware.md) §11.2–11.3 |
