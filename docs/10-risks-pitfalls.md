# VEKTOR — Risk Register & Known Engineering Pitfalls

[← Back to VEKTOR-PRD.md](../VEKTOR-PRD.md)

---

## 16. Risk Register

| ID | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| R-001 | GPU supply constraints delay hardware procurement | Medium | High | Order H100s 6+ months ahead; qualify AMD MI300X as fallback |
| R-002 | LLM hallucinations produce dangerous COA recommendations | Medium | Critical | Human-in-the-loop mandatory; Zod schema gates all LLM output; confidence threshold blocks low-quality responses |
| R-003 | `onnxruntime-node` GPU build fails on target hardware | High | High | Test GPU CUDA build in CI against target OS; maintain CPU fallback path; pin version |
| R-004 | `node-llama-cpp` OOM on Jetson unified memory | High | High | Q4_K_M quantization mandatory; memory budget enforced in CI (`vektor-edge`) |
| R-005 | Sensor timestamp skew causes phantom tracks | High | High | `sensor_ts` (GPS) used for all ordering — never `kafka_received_at`; 500ms watermark |
| R-006 | Kafka topic schema drift breaks consumers | Medium | Medium | Zod/proto schemas centralized in `vektor-proto`; contract PR required for any change (§9.4) |
| R-007 | `llm-cloud-svc` (70B model) exceeds GPU VRAM budget | Medium | High | Quantize to AWQ/GPTQ 4-bit; tensor parallel across 2x H100; fallback to 13B model |
| R-008 | DeepSORT TypeScript re-implementation diverges from reference | Medium | Medium | Unit test against reference Python outputs; parity test suite in CI |
| R-009 | Single points of failure in Kafka or PostgreSQL | Medium | High | 3-broker Kafka (ISR=2); PostgreSQL HA via Patroni; weekly failover drills |
| R-010 | Polyrepo overhead slows cross-cutting changes | Medium | Medium | `vektor-proto` versioning + generated stubs keep cross-repo changes mechanical, not manual; document the "change touches N repos" runbook |
| R-011 | Key engineers depart mid-project | Low | High | ADRs for every major decision; pair programming; no single points of human failure |

---

## 17. Known Engineering Pitfalls

> **⚠️ PITFALL 1 — Data Synchronization Bottleneck**
>
> Do not use `kafka_received_at` for sensor data ordering. Sensors transmit out-of-order due to variable network latency. If the fusion engine processes events by arrival time, moving vehicles generate disjointed phantom tracks — the same vehicle appears at two locations simultaneously.
>
> **Mitigation:** `fusion-svc`'s Kafka consumer must order events by `sensor_ts` (GPS-derived, NTP-synced). Apply a 500ms event-time watermark in the Redis Streams windowing logic. Enforced by the `DetectionEvent` schema (defined in `vektor-proto`), which validates `sensor_ts` and rejects malformed events at the consumer boundary.

---

> **⚠️ PITFALL 2 — Edge Inference Memory Starvation**
>
> Standard FP32 ONNX models will OOM the Jetson AGX Orin. YOLOv8m at FP32 needs ~4.5 GB GPU memory. The Jetson has unified CPU/GPU memory — inference alongside the local DB, BullMQ, and sync workers will exhaust the 64 GB budget.
>
> **Mitigation:** All edge ONNX models must be INT8-quantized. Target < 1.5 GB per model. A CI step in `vektor-edge` rejects any ONNX model exceeding budget. Never ship FP32 weights to the edge image.

---

> **⚠️ PITFALL 3 — Cross-Repo Contract Drift**
>
> As `vektor-backend` and `vektor-ml` evolve independently, it's tempting to hand-edit a `.proto` copy locally instead of going through `vektor-proto`. This silently reintroduces the exact TS/Python drift problem the repo split was meant to prevent.
>
> **Mitigation:** `vektor-backend` and `vektor-ml` never vendor `.proto` files directly — both consume only the published `@vektor/proto` / `vektor-proto` packages, pinned by version. CI in both repos fails the build if a local `.proto` file is detected outside `vektor-proto`. Contract changes require a PR in `vektor-proto` with dual approval (§9.9) before either consuming repo can bump its dependency.

---

> **⚠️ PITFALL 4 — LLM Prompt Injection via Sensor Metadata**
>
> Sensor metadata (names, tags, analyst notes) flows into the COA prompt context. A crafted sensor name or operator note could inject instructions into the LLM, producing dangerous or nonsensical COA outputs.
>
> **Mitigation:** `coa-svc`'s context builder sanitizes all string fields via `sanitizeForPrompt()` before assembling the prompt. Context is passed as structured JSON — never free-form text. All LLM responses are validated through a strict `COAResponse` schema before touching application logic; responses failing validation are rejected and logged.

---

> **⚠️ PITFALL 5 — `onnxruntime-node` GPU Build Fragility**
>
> `onnxruntime-node` has a native CUDA binding that must match the exact CUDA toolkit version on the host. A mismatch causes a silent fallback to CPU inference — reducing throughput by 10–50x without any obvious error.
>
> **Mitigation:** Pin `onnxruntime-node` to a specific version. Document the exact CUDA toolkit version it was compiled against. Add a startup health check in `cv-inference-svc` that confirms the GPU execution provider is active — fail fast if CPU-only mode is detected. Test the full Docker image against the target CUDA version in CI before every release.

---

> **⚠️ PITFALL 6 — `node-llama-cpp` Memory Leak on Long-Running Sessions**
>
> `node-llama-cpp` maintains a native C++ context object per inference session. On edge deployments running 24/7, failing to explicitly dispose the context/model leads to a slow native memory leak that crashes the Node.js process after several hours.
>
> **Mitigation:** Always wrap `node-llama-cpp` calls in a `try/finally` that disposes the context. Implement a BullMQ worker (not a long-lived in-process loop) for COA generation so each job gets a fresh context. Add a memory health-check endpoint in `coa-svc` that alerts if RSS exceeds threshold.

---

