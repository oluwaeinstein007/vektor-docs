# VEKTOR — Vision — Naming, Summary, Problem, Goals

[← Back to VEKTOR-PRD.md](../VEKTOR-PRD.md)

---

## 0. Naming Decision

### 0.1 Why not ARES

"ARES" (Advanced Real-time Electronic Surveillance) has three problems: it's a Greek god-of-war name shared by dozens of unrelated defense products (name collisions, weak trademark position), the backronym is generic ("surveillance" undersells the fusion/decision-support core of the product), and the militaristic framing actively fights the product's primary civilian go-to-market (disaster response, infrastructure monitoring, maritime, logistics).

### 0.2 Candidates considered

| Name | Verdict |
|---|---|
| AEGIS-anything | Rejected — already a real, famous DoD combat system (Aegis Combat System); guaranteed confusion |
| SENTRY / NEXUS / MERIDIAN | Rejected — generic SaaS-brand words, near-zero trademark distinctiveness, no product story |
| ATLAS | Rejected — used as the working codename for the parallel draft, but it's an extremely common product name (AWS, MongoDB Atlas, Palantir-adjacent tools already use it) |
| **VEKTOR** | **Selected** |

### 0.3 Why VEKTOR

**Visual Edge & Kinematic Tracking Operations Runtime.**

- **Visual** — the CV/detection pipeline (YOLO, ONNX inference) is the sensing core.
- **Edge** — 72-hour offline edge deployment is a first-class, not bolted-on, capability.
- **Kinematic Tracking** — position/velocity/heading state estimation (the EKF/DeepSORT layer) is the product's technical differentiator.
- **Operations Runtime** — it's infrastructure operators run their C2 workflow on, not a point tool.

The deliberate misspelling of "Vector" (a word that is otherwise entirely descriptive and unregisterable on its own — the system is, literally, tracking kinematic vectors and storing embedding vectors) gives a distinctive, trademarkable wordmark while keeping the semantic hook: **every core subsystem — position vectors, velocity vectors, RF bearing vectors, embedding vectors in the RAG store — is, mathematically, a vector.** The name is the product.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Problem Statement](#2-problem-statement)
3. [Goals & Success Metrics](#3-goals--success-metrics)
4. [Stakeholders & Team](#4-stakeholders--team)
5. [User Types](#5-user-types)
6. [Scope & Boundaries](#6-scope--boundaries)
7. [Functional Requirements (Epics)](#7-functional-requirements-epics)
8. [Non-Functional Requirements](#8-non-functional-requirements)
9. [System Design & Repository Architecture](#9-system-design--repository-architecture)
10. [Tech Stack](#10-tech-stack)
11. [Hardware Architecture & Deployment](#11-hardware-architecture--deployment)
12. [Data Architecture & Core Models](#12-data-architecture--core-models)
13. [API Contracts](#13-api-contracts)
14. [Security, Compliance & Data Classification](#14-security-compliance--data-classification)
15. [Development Roadmap](#15-development-roadmap)
16. [Risk Register](#16-risk-register)
17. [Known Engineering Pitfalls](#17-known-engineering-pitfalls)
18. [Cost Model & Team Composition](#18-cost-model--team-composition)
19. [Glossary](#19-glossary)

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

## 2. Problem Statement

### 2.1 Current State Pain Points

| Pain Point | Impact |
|---|---|
| Operators manage 8–12 disconnected screens and feeds | Cognitive overload; missed threats; slow decisions |
| Targeting/routing data takes 12+ hours to propagate through workflows | Outdated picture; stale decisions |
| No correlation engine — the same object is reported by multiple sensors as different entities | Phantom tracks; wasted analyst effort |
| No offline capability at the tactical/field edge | Complete blackout when connectivity drops |
| LLM/AI tools in use today are external, cloud-dependent, and data-leaking | Security violations; unusable in sensitive or air-gapped environments |
| ERPs and logistics systems are disconnected from the live operational picture | Stock-outs, misrouted convoys, no predictive resupply |

### 2.2 Desired Future State

A single operator console that fuses all sensor inputs in real time, automatically detects and tracks objects, correlates cross-domain data into unified entity records, and surfaces ranked, human-approved response recommendations — all deployable on-prem or at the tactical/field edge with zero cloud dependency.

---

## 3. Goals & Success Metrics

### 3.1 Primary Goals

| # | Goal | Metric | Target |
|---|---|---|---|
| G1 | Compress decision loop | Sensor-to-decision latency | < 60 seconds |
| G2 | Unify operator interface | Number of screens in active use | 1 (from 8–12) |
| G3 | High-confidence detection | Object detection accuracy | > 85% mAP |
| G4 | Real-time data fusion | Track deduplication rate | > 95% |
| G5 | Reliable edge operation | Uptime without cloud | 72 hours continuous |
| G6 | Fast ingestion | Sensor-to-normalization latency | < 50ms |
| G7 | Operator throughput | Targets processed per operator per hour | 10x baseline |
| G8 | Interoperability | Third-party integrations live via public API | ≥ 3 by GA |

### 3.2 Anti-Goals (What VEKTOR Does NOT Do)

- VEKTOR does not autonomously execute any response action — all actions require human approval. **Permanently out of scope, not a v1 limitation.**
- VEKTOR does not replace battlefield/dedicated comms systems (e.g., ATAK, LINK-16).
- VEKTOR does not perform open-internet web browsing or external data lookups at inference time.
- VEKTOR does not store personally identifiable information (PII) of civilians beyond what's operationally necessary and retention-governed (§12.4, §14).
- VEKTOR does not call any external/third-party AI API (OpenAI, Anthropic, etc.) — all inference is self-hosted, to guarantee no data leaves the deployment boundary.

---

