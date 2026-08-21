# VEKTOR — Glossary

[← Back to VEKTOR-PRD.md](../VEKTOR-PRD.md)

---

## 19. Glossary

| Term | Definition |
|---|---|
| **ADS-B** | Automatic Dependent Surveillance–Broadcast; aircraft position transponder system |
| **AIS** | Automatic Identification System; maritime vessel tracking |
| **@vektor/proto** | Published npm package of generated TS stubs from `vektor-proto` |
| **BullMQ** | Redis-backed job queue for Node.js; the primary stateful-worker system |
| **C2** | Command and Control |
| **CJADC2** | Combined Joint All-Domain Command and Control (reference term for the broader C2 category) |
| **COA** | Course of Action; a ranked recommended response option |
| **COP** | Common Operational Picture; unified situational awareness display |
| **CV** | Computer Vision |
| **DIL** | Disconnected, Intermittent, and Limited (network environment) |
| **Drizzle ORM** | TypeScript-native ORM with PostGIS and TimescaleDB support |
| **EKF** | Extended Kalman Filter; algorithm for nonlinear state estimation — implemented in TypeScript via mathjs |
| **EW** | Electronic Warfare; RF emissions, jamming, signals intelligence |
| **FMV** | Full-Motion Video; continuous live video from drones or cameras |
| **GeoTIFF** | Georeferenced TIFF imagery format with embedded coordinate metadata |
| **gRPC** | Google Remote Procedure Call; the only protocol crossing the TS↔Python boundary |
| **HPA** | Horizontal Pod Autoscaler (Kubernetes) |
| **INT8** | 8-bit integer quantization; mandatory for all edge-deployed ONNX models |
| **kafkajs** | Pure TypeScript Apache Kafka client |
| **MGRS** | Military Grid Reference System; tactical coordinate system |
| **mAP** | Mean Average Precision; object detection accuracy metric |
| **mTLS** | Mutual TLS; both client and server authenticate |
| **node-llama-cpp** | Node.js native binding for llama.cpp; runs quantized LLMs entirely in TypeScript |
| **NTP/PTP** | Network Time Protocol / Precision Time Protocol; clock sync for sensor alignment |
| **ONNX** | Open Neural Network Exchange; the bridge format between Python training and TS inference |
| **onnxruntime-node** | Node.js binding for ONNX Runtime; enables GPU-accelerated inference in TypeScript |
| **RAG** | Retrieval-Augmented Generation; LLM technique using external knowledge retrieval |
| **RBAC** | Role-Based Access Control |
| **SAR** | Synthetic Aperture Radar |
| **SRT** | Secure Reliable Transport; low-latency video streaming protocol |
| **Turborepo** | Monorepo build system managing task pipelines within `vektor-platform` |
| **vLLM** | Python library for high-throughput LLM serving; used only inside `vektor-ml` |
| **@xenova/transformers** | Hugging Face Transformers.js; runs ONNX embedding models in Node.js |
| **WGS 84** | World Geodetic System 1984; global coordinate reference system |
| **Zod** | TypeScript-first schema validation library; enforces type safety at all service boundaries and LLM outputs |

---

*VEKTOR PRD v1.0.0 — August 2026. Merges and supersedes ARES-PRD-v2.md, ARES-PRD-v3.md, and maven-smart-system-prd.md (ATLAS). TypeScript-first implementation; Python strictly isolated to the `vektor-ml` repo. All specifications subject to engineering review and executive sign-off prior to implementation.*
