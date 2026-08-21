# VEKTOR — Hardware Architecture & Deployment

[← Back to VEKTOR-PRD.md](../VEKTOR-PRD.md)

---

## 11. Hardware Architecture & Deployment

### 11.1 Deployment Tiers

| Tier | Profile | Language Runtime |
|---|---|---|
| **Cloud Core** | Full K8s cluster, H100 GPUs, petabyte storage | Node.js (TS) + isolated Python vLLM container |
| **On-Premises DC** | 2-rack server room, same stack | Node.js (TS) + isolated Python vLLM container |
| **Tactical / Field Edge Node** | Ruggedized Jetson AGX Orin, k3s, 72h offline | Node.js (TS) only — `node-llama-cpp` + `onnxruntime-node` |

> **Edge is 100% TypeScript.** No Python runtime on edge nodes.

### 11.2 Cloud Core / Data Center — Compute

| Node Type | Specification | Count | Purpose |
|---|---|---|---|
| GPU — Inference + LLM | 2x AMD EPYC 9654 (96c), 1.5 TB DDR5, 8x NVIDIA H100 PCIe 80GB | 2–4 | `onnxruntime-node` CV + Python vLLM LLaMA 70B |
| General Compute | 16 vCPU, 64 GB RAM | 6 | Fastify APIs, Kafka, Redis, fusion workers |
| Memory-Optimized | 32 vCPU, 256 GB RAM | 2 | Kafka brokers, Elasticsearch |
| Storage-Optimized | 16 vCPU, 64 GB RAM, 8 TB NVMe | 4 | TimescaleDB, MinIO, Qdrant |
| K8s Control Plane | 3x dual Xeon, 64 GB RAM | 3 | etcd + API server (HA) |

**Storage:**

| Type | Capacity | Technology |
|---|---|---|
| Hot imagery | 100 TB | NVMe SSD (Ceph / MinIO, > 10 GB/s write) |
| Cold archive | 1 PB+ | HDD (MinIO tiered) |
| Database volumes | 10 TB | NVMe SSD, high IOPS |
| Redis cache | 512 GB | In-memory, NVMe-backed AOF |

**Network:** internal backbone 100 GbE (dual NIC/node); GPU interconnect NVLink + InfiniBand HDR 400 Gbps; external uplink 10 Gbps min; pod-to-pod latency < 1ms.

### 11.3 On-Premises Server Rack Layout

**Rack A — Compute (42U)**

| Slots | Hardware | Spec |
|---|---|---|
| 1–4 | GPU Servers (CV + LLM) | Dell PowerEdge R750xa, 2x NVIDIA H100 80GB, AMD EPYC 9654, 1.5 TB DDR5 |
| 5–8 | App / API Servers (Node.js TS) | Dell PowerEdge R650, 2x Xeon Gold 6330, 256 GB DDR4 |
| 9–12 | Kafka + Redis Cluster | Dell PowerEdge R650, 512 GB DDR4 |
| 13–15 | K8s Control Plane (HA) | 3x Dell PowerEdge R450 |
| 16–18 | Load Balancers | 2x F5 BIG-IP i4800 |
| 19–22 | Core Switches | Cisco Nexus 9300 (100 GbE) |
| 23–26 | Firewall | Palo Alto PA-5450 (HA pair) |
| 27–30 | OOB Management | Dell iDRAC Enterprise |

**Rack B — Storage & Data (42U)**

| Slots | Hardware | Spec |
|---|---|---|
| 1–6 | NVMe Storage Arrays | Dell PowerStore 1200T (2x 100 TB NVMe) |
| 7–12 | HDD Cold Storage | Dell PowerVault ME5024 (2x 500 TB) |
| 13–16 | Database Servers | 4x Dell PowerEdge R740xd, 512 GB ECC RAM |
| 17–20 | Redis Cluster | 4x Supermicro, 1.5 TB DRAM total |
| 21–24 | Backup Appliances | Veeam on PowerEdge R450 |
| 25–28 | UPS + PDUs | APC Smart-UPS Ultra 20 kW x2 |

**Power & Cooling:** ~80–100 kW peak draw; 200 kVA UPS (N+1); hot-aisle containment CRAC units.

### 11.4 Tactical Edge Node (100% TypeScript Runtime)

| Component | Hardware | Specification |
|---|---|---|
| **Edge AI Server** | NVIDIA Jetson AGX Orin Industrial (x2) | 64-core ARM Cortex-A78AE, 64 GB unified memory, 2048-core Ampere GPU, 275 TOPS |
| **Tactical Server** | Intel NUC 13 Pro (rugged) | 16-core i9, 64 GB RAM, 2 TB NVMe |
| **Storage** | Samsung T7 Shield SSD x4 | 4 TB total, AES-256 hardware encryption |
| **Network** | Peplink MAX Transit Pro | Dual 5G/LTE, Starlink fallback, SD-WAN |
| **Switch** | Cisco IE-3400 Industrial | -40°C to 75°C, IP67 |
| **Power** | Victron MultiPlus 24V/3000W | Solar + LiPo battery + shore/generator |
| **Display** | Xenarc 1000TSH 10" | 1000 nit, IP65, glove-touch |
| **VPN** | WireGuard (hardware-accelerated) | AES-256 tunnel to cloud core |

**Edge Software Stack (Node.js / TypeScript only):**

| Component | Library | Memory Budget |
|---|---|---|
| CV Inference | `onnxruntime-node` + YOLOv8n INT8 ONNX | < 1.5 GB GPU |
| Local LLM | `node-llama-cpp` + Mistral 7B Q4_K_M | < 6 GB unified memory |
| Embeddings | `@xenova/transformers` (MiniLM ONNX) | < 256 MB |
| Offline DB | `better-sqlite3` + local PostGIS-lite queries | < 512 MB |
| Map Tiles | PMTiles static server (TS) | < 100 MB RAM |
| Sync Queue | BullMQ + `ioredis` (local Redis) | < 256 MB |
| Runtime | Node.js 22 LTS (ARM64) | — |

**Total estimated edge memory footprint: < 10 GB** (fits within Jetson AGX Orin's 64 GB unified memory with headroom for OS + networking).

### 11.5 End-User Devices

| Device | Spec | Assigned Role |
|---|---|---|
| Operator Workstation | Dell Precision 5860, 64 GB RAM, NVIDIA RTX 4090, dual 4K | HQ commanders, analysts |
| Tactical Tablet | Getac ZX10, Android 12, IP67 | Field supervisors |
| Ruggedized Laptop | Panasonic Toughbook 55, i7, 32 GB | Forward analysts |
| Field Phone | Samsung Galaxy XCover7 (IP68) | Field operators |

---

