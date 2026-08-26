did di # VEKTOR — Visual Edge & Kinematic Tracking Operations Runtime

## Master Product Requirements & System Design Document

> **Formerly:** ARES (Advanced Real-time Electronic Surveillance)

| Field                | Value                                                                   |
| -------------------- | ----------------------------------------------------------------------- |
| **Document Version** | v1.0.0 (Master)                                                         |
| **Status**           | In Review                                                               |
| **Classification**   | Internal — Confidential                                                 |
| **Date**             | August 2026                                                             |
| **Author(s)**        | Engineering Leadership                                                  |
| **Reviewers**        | CTO, Product, Security Lead, ML Lead                                    |
| **Supersedes**       | `ARES-PRD-v2.md`, `ARES-PRD-v3.md`, `maven-smart-system-prd.md` (ATLAS) |

### Revision History

| Version           | Date         | Summary                                                                                                                                                                                              |
| ----------------- | ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ARES v1.0.0       | May 2026     | Initial draft                                                                                                                                                                                        |
| ARES v2.0.0       | Jun 2026     | Full expansion: personas, NFRs, data models, API contracts, risk register                                                                                                                            |
| ARES v3.0.0       | Jun 2026     | TypeScript-first stack decision; Python quarantined to 2 services                                                                                                                                    |
| ATLAS v1.0.0      | Jun 2026     | Parallel civilian-framing draft ("Maven Smart System clone"); richer feature specs, cost model, team plan                                                                                            |
| **VEKTOR v1.0.0** | **Aug 2026** | **Merge of all three documents into one canonical spec. Rebrand ARES → VEKTOR. Adds §9 System Design with a 6-repo polyrepo architecture, contract-boundary repo, and repo-to-repo CI/CD topology.** |
| **VEKTOR v1.0.1** | **Aug 2026** | **§9's `vektor-platform` split into `vektor-backend` (services + shared packages) and `vektor-web` (the frontend, `apps/web` and the not-yet-built `field-pwa`) — now a 7-repo polyrepo. `vektor-web` consumes `@vektor/shared` from `vektor-backend` via the same cross-repo `link:` pattern already used for `vektor-proto`.** |

---

## 0. Naming Decision

### 0.1 Why not ARES

"ARES" (Advanced Real-time Electronic Surveillance) has three problems: it's a Greek god-of-war name shared by dozens of unrelated defense products (name collisions, weak trademark position), the backronym is generic ("surveillance" undersells the fusion/decision-support core of the product), and the militaristic framing actively fights the product's primary civilian go-to-market (disaster response, infrastructure monitoring, maritime, logistics).

### 0.2 Candidates considered

| Name                      | Verdict                                                                                                                                                                |
| ------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| AEGIS-anything            | Rejected — already a real, famous DoD combat system (Aegis Combat System); guaranteed confusion                                                                        |
| SENTRY / NEXUS / MERIDIAN | Rejected — generic SaaS-brand words, near-zero trademark distinctiveness, no product story                                                                             |
| ATLAS                     | Rejected — used as the working codename for the parallel draft, but it's an extremely common product name (AWS, MongoDB Atlas, Palantir-adjacent tools already use it) |
| **VEKTOR**                | **Selected**                                                                                                                                                           |

### 0.3 Why VEKTOR

**Visual Edge & Kinematic Tracking Operations Runtime.**

- **Visual** — the CV/detection pipeline (YOLO, ONNX inference) is the sensing core.
- **Edge** — 72-hour offline edge deployment is a first-class, not bolted-on, capability.
- **Kinematic Tracking** — position/velocity/heading state estimation (the EKF/DeepSORT layer) is the product's technical differentiator.
- **Operations Runtime** — it's infrastructure operators run their C2 workflow on, not a point tool.

The deliberate misspelling of "Vector" (a word that is otherwise entirely descriptive and unregisterable on its own — the system is, literally, tracking kinematic vectors and storing embedding vectors) gives a distinctive, trademarkable wordmark while keeping the semantic hook: **every core subsystem — position vectors, velocity vectors, RF bearing vectors, embedding vectors in the RAG store — is, mathematically, a vector.** The name is the product.

---

## Document Map

This file is the entry point: naming rationale and the executive summary live here in full. Everything else has been split into `docs/` so each concern can be read, linked, and reviewed on its own — see [`docs/README.md`](docs/README.md) for the same map in one place. Sections 0–1 stay inline below; the rest are one click away.

| #     | Section                                               | File                                                               |
| ----- | ----------------------------------------------------- | ------------------------------------------------------------------ |
| 0     | Naming Decision                                       | _(inline below)_                                                   |
| 1     | Executive Summary                                     | _(inline below)_                                                   |
| 2–3   | Problem Statement · Goals & Success Metrics           | [`docs/00-vision.md`](docs/00-vision.md)                           |
| 4–6   | Stakeholders & Team · User Types · Scope & Boundaries | [`docs/01-users-scope.md`](docs/01-users-scope.md)                 |
| 7     | Functional Requirements (Epics)                       | [`docs/02-requirements.md`](docs/02-requirements.md)               |
| 8     | Non-Functional Requirements                           | [`docs/03-nfr.md`](docs/03-nfr.md)                                 |
| 9     | System Design & Repository Architecture               | [`docs/04-system-design.md`](docs/04-system-design.md)             |
| 10    | Tech Stack                                            | [`docs/05-tech-stack.md`](docs/05-tech-stack.md)                   |
| 11    | Hardware Architecture & Deployment                    | [`docs/06-hardware.md`](docs/06-hardware.md)                       |
| 12–13 | Data Architecture · API Contracts                     | [`docs/07-data-api.md`](docs/07-data-api.md)                       |
| 14    | Security, Compliance & Data Classification            | [`docs/08-security-compliance.md`](docs/08-security-compliance.md) |
| 15    | Development Roadmap                                   | [`docs/09-roadmap.md`](docs/09-roadmap.md)                         |
| 16–17 | Risk Register · Known Engineering Pitfalls            | [`docs/10-risks-pitfalls.md`](docs/10-risks-pitfalls.md)           |
| 18    | Cost Model & Team Composition                         | [`docs/11-cost-team.md`](docs/11-cost-team.md)                     |
| 19    | Glossary                                              | [`docs/12-glossary.md`](docs/12-glossary.md)                       |

> Note on §-references inside the split files (e.g. "see §9.4"): these refer to the original numbered sections above, not file names — cross-file, they're a pointer to _which_ doc to open, not a clickable anchor.

---

## 1. Executive Summary

**VEKTOR** is a next-generation Command & Control (C2) platform, delivered as both a cloud SaaS and a ruggedized edge deployment, that solves one core problem: **data inundation kills decision speed.**

Operators today juggle 8–12 disconnected screens — video feeds, radar outputs, AIS/ADS-B trackers, ERP dashboards, radio channels — and manually stitch them into a mental picture. VEKTOR collapses all of that into one AI-enriched interface by:

- **Ingesting** live feeds from drones, satellites, SAR, EW/RF signals, AIS ships, ADS-B aircraft — all simultaneously, normalized to a common spatial/temporal schema.
- **Seeing** — running real-time computer vision (YOLOv8) to detect, classify, and track objects across every video feed automatically.
- **Fusing** — correlating the same object seen by multiple sensors into one unified entity record, eliminating phantom duplicates.
- **Deciding** — generating ranked Courses of Action (COA) via a self-hosted LLM, matched against available assets and no-strike zones, for a human commander to approve or reject.
- **Surviving** — running fully offline on a ruggedized edge node (Jetson AGX Orin) for 72 hours with zero cloud dependency.

The deterministic recommendation engine compresses the sensor-to-decision loop from hours to under 60 seconds.

**Language strategy:** VEKTOR is built on a **TypeScript-first** codebase (~90%+ of all services, APIs, stream processing, frontend, and edge runtime). Python is strictly quarantined to two isolated, containerized, gRPC-only services: the CV model training pipeline and the cloud-tier LLM server. No other service imports Python code or shares a runtime with it. See §9 for how this boundary is enforced structurally, not just by convention.

**Civilian applications:** disaster response coordination, critical infrastructure monitoring, maritime surveillance, large-scale logistics operations, smart-city operations centers.

---

See the [Document Map](#document-map) above for Problem Statement, Goals, Personas, Requirements, System Design, and everything else.

---

_VEKTOR PRD v1.0.0 — August 2026. Merges and supersedes ARES-PRD-v2.md, ARES-PRD-v3.md, and maven-smart-system-prd.md (ATLAS). Full content now split across this file and `docs/` — see `docs/README.md` for the complete map._
