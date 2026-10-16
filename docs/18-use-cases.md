# VEKTOR — Use Cases & Target Markets

[← Back to VEKTOR-PRD.md](../VEKTOR-PRD.md) · [Current System Status](16-system-status.md) · [Next Work](17-next-work.md) · [Commercialization](19-commercialization.md)

---

*Written 2026-10-16. The capability inventory below reflects verified code as of that date; the market analysis is reasoning from that inventory, not from customer research — see "What this document is not" at the end.*

## What VEKTOR can actually do today

Before picking markets, it's worth stating plainly what exists, because the honest inventory is both shorter and more impressive than the PRD's framing suggests.

| Capability | Carried by | State |
|---|---|---|
| Ingest live video (RTSP/SRT) and extract frames | `ingest-svc` | Verified against MediaMTX and a real Android phone |
| Ingest satellite/SAR imagery (GeoTIFF) | `ingest-svc` | Verified against real generated GeoTIFF |
| Ingest ship traffic (AIS) | `ingest-svc` | Verified against the live Norwegian coastal feed |
| Ingest aircraft traffic (ADS-B) | `ingest-svc` | Verified against a real SBS-1 stream |
| Ingest arbitrary IoT sensors (MQTT) | `ingest-svc` | Verified against Mosquitto |
| Ingest drone/UGV telemetry (MAVLink) | `mavlink-bridge` | Verified against real/SITL MAVLink |
| Ingest phone GPS/gyro/camera from the field | `ingest-svc` + `field-pwa` | Verified in a real browser |
| Detect and classify objects in video | `cv-inference-svc` | Real YOLOv8 via onnxruntime-node, CPU-verified |
| Track objects with persistent IDs across frames | `cv-inference-svc` | DeepSORT re-implemented in TypeScript |
| Train a detector on custom objects, export INT8 ONNX | `cv-train-svc` | Real end-to-end training run verified |
| Hot-swap a detection model without downtime | `cv-inference-svc` | Verified with two genuinely different models |
| Fuse the same object seen by several sensors into one entity | `fusion-svc` | Real 6-state EKF, geodetic, verified |
| Flag friendly assets and auto-buffer no-strike zones | `fusion-svc` | Meters-accurate via PostGIS geography |
| Generate ranked courses of action from an LLM, grounded in doctrine | `coa-svc` + `packages/qdrant` | RAG over Qdrant + BGE-M3, human approval required |
| Fire geofence and anomaly alerts to email/webhook/SMS | `alert-svc` | Real EWMA/z-score detector, real dispatch |
| Sync inventory from an ERP and forecast depletion | `logistics-svc` | Real Debezium CDC, real linear regression |
| Compute routes that avoid hazard zones | `logistics-svc` | Real pgRouting `pgr_dijkstra` |
| Generate PDF/PPTX situation reports on a schedule | `reporting-svc` | Real Puppeteer and pptxgenjs output |
| Keep an append-only, RLS-protected audit log | `audit-svc` | Postgres row-level security verified |
| Run fully offline for 72 hours and resync on reconnect | `edge-sync-svc` + `vektor-edge` | Offset-persisted Kafka replay, IndexedDB queue |
| Serve maps with no internet connection | `map-tile-server` + PMTiles packs | Real tippecanoe→pmtiles pipeline |
| Show all of it on one live map with role-gated screens | `apps/web` | MapLibre, Socket.io, RBAC-gated navigation |

Three things about this list matter commercially more than any individual row.

**It is domain-neutral.** Nothing above is specific to defence. An EKF does not know whether it is tracking a frigate or a delivery truck. The military framing is a choice the PRD made, not a property of the code — and §0.1 already recorded that the original "ARES" name was rejected precisely because militaristic framing fought the civilian go-to-market.

**The offline capability is unusual.** Most surveillance and situational-awareness platforms assume connectivity. A genuine 72-hour disconnected mode with delta resync is hard, it's built, and it's the thing competitors will struggle to match quickly.

**Self-hosted inference is a feature, not a limitation.** The anti-goal "VEKTOR does not call any external AI API" reads as a constraint in the PRD. In a sales conversation with a utility, a port authority or a government agency, "no data leaves your boundary, ever, including to the AI" is the sentence that ends the security objection.

---

## Eight markets, honestly ranked

Each section below gives the scenario, what already carries it, what's genuinely missing, who signs, and what makes the sale hard.

### A. Maritime domain awareness — *strongest near-term fit*

**Scenario.** A port authority, coastal agency or offshore operator watches its waters. AIS gives cooperative traffic; cameras and radar catch the vessels that have switched their transponders off. VEKTOR fuses both into one picture, flags a vessel that enters a restricted anchorage, loiters near a cable route, or goes dark mid-transit, and fires an alert with a recommended response.

**Already carried.** This is the vertical VEKTOR is closest to serving today. The AIS adapter has been run against a real public coastal feed — not synthetic data. Geofence zones, the EWMA anomaly detector, no-strike/protected-zone buffering, PDF sitreps and the live map all apply unchanged. AIS-dark detection is literally "an entity whose AIS track stops while a camera track continues", which is exactly what the fusion engine already does.

**Missing.** Historical playback (`NW-018`) — incident review is most of the job in this vertical. Radar ingest, which is a connector (`NW-019`), not an architecture change.

**Buyer.** Port authority operations director; coastal agency; offshore wind or subsea cable operator. One real published anchor exists for deal size: Sirius Insight lists maritime domain situational awareness on the UK G-Cloud framework at **£100,000 per licence per year**. That is a platform price for an agency, not a per-device licence — which is the shape VEKTOR's Operations Platform tier should take. Notably, no other maritime vendor found publishes pricing at all.

**Why VEKTOR.** Incumbents sell AIS-only dashboards or camera-only VMS. Fusing the two with persistent cross-sensor identity is the differentiator, and the civilian framing means none of the export-control drag of the defence market.

### B. Critical infrastructure and perimeter monitoring — *strongest commercial fit*

**Scenario.** A substation, pipeline right-of-way, water treatment plant, data centre campus or solar farm. Dozens of fixed cameras, some drone patrols, a SCADA system. VEKTOR detects a person or vehicle inside a perimeter, tracks them across camera handoffs as one entity rather than six disconnected alerts, correlates with the SCADA tap, and escalates with evidence attached.

**Already carried.** CV detection, cross-camera tracking, geofences, alert dispatch, scheduled reporting, the audit log, RBAC. Custom model training (`cv-train-svc`) directly addresses the real complaint about existing systems — generic detectors fire on deer and shadows.

**Missing.** ONVIF camera discovery (`NW-019`) — infrastructure operators have hundreds of existing cameras and will not re-cable. Historical playback (`NW-018`) for incident packets. Multi-site rollup if you sell to an operator with many sites.

**Buyer.** Utility or operator security director, with a compliance mandate that funds the line item. That claim is now backed rather than asserted: under NERC CIP-014, utilities reported **$10–15M** in substation physical-security spending in NERC's own response to a FERC order, the E-ISAC logged ~**1,700 physical security incidents in 2022** (up 10.5% year on year), and violations reportedly carry fines up to $1M per day. Mandated, incident-driven spend is the shape that closes fastest. See [19-commercialization.md](19-commercialization.md) for sources and the pricing anchors.

**Why VEKTOR.** This is the market where "the AI runs inside your network, nothing leaves" is not a nice-to-have — OT security teams will not permit cloud video analytics on operational networks. The self-hosted anti-goal is a direct competitive weapon here, and the edge deployment mode means remote unmanned sites work without backhaul.

### C. Disaster response coordination — *strongest mission fit, hardest to monetise*

**Scenario.** Flood, wildfire, earthquake. Communications are down. An incident commander has drone imagery, field teams with phones, supply depots, and no connectivity. An edge node in a vehicle runs the full picture locally for days, field teams submit spot reports that queue and sync when a link appears, and the logistics layer routes supplies around impassable roads.

**Already carried.** This is what the edge profile was built for, and it's complete: 72-hour offline, offline PMTiles maps, `field-pwa` for GPS/photo spot reports from any phone, IndexedDB queueing with delta resync, pgRouting with blocked-edge avoidance, depletion forecasting.

**Missing.** Nothing architectural. Hardware validation (`NW-012`) matters most here because the edge node is the product.

**Buyer.** Emergency management agency, national disaster authority, large NGO. Budgets are real but slow, episodic and grant-shaped; procurement is often annual and political.

**Why VEKTOR.** Almost nothing else in this space works disconnected. But treat this as the vertical that wins credibility and press rather than the one that pays early bills — and note that a disaster-response reference customer makes every other vertical easier to sell.

### D. Border, wide-area and remote-site surveillance

**Scenario.** A long perimeter or a remote facility with towers, sensors and intermittent connectivity.

**Already carried.** Nearly all of it — this is essentially vertical B at larger geographic scale with vertical C's connectivity assumptions.

**Missing.** Scale validation (`NW-014`: 1,000+ tracks has never been tested) and ground-radar connectors.

**Buyer.** Government agency, large private landholder, mining or energy operator with remote sites. Mixed: the private-sector half of this market is far easier than the government half.

### E. Drone and UAS operations

**Scenario.** An organisation operating a drone fleet — inspection, survey, security patrol — needs one picture of where every aircraft is, what each one is seeing, and whether anything uncooperative is in the airspace.

**Already carried.** `mavlink-bridge` already fuses drone telemetry as a friendly track through the same contract as any other sensor. ADS-B gives cooperative manned traffic. CV runs on the drone's own video feed. The blue-force/no-strike machinery maps directly onto "our aircraft" versus "protected airspace".

**Missing.** Remote ID ingest (a connector), and airspace-authority integration if you go near regulated BVLOS operations.

**Buyer.** Drone service provider, infrastructure inspection firm, large site operator. Smaller deals, much faster cycles, and a good first-revenue candidate precisely because nobody's security team needs to approve a six-month review.

### F. Smart city and large-event operations

**Scenario.** A stadium, festival, transit hub or city operations centre. Crowd density, vehicle flow, incident detection across many camera feeds, with one coordinated response picture.

**Already carried.** Detection, tracking, geofencing, alerting, multi-feed fusion, the operator dashboard.

**Missing.** Crowd-density estimation specifically (a model, trainable via `cv-train-svc`, not a code change), and — seriously — a privacy posture. The PRD's PII anti-goal is a start but a European or Californian city will require a documented data-protection impact assessment, retention policy and face-handling stance before a procurement conversation begins.

**Buyer.** City operations, stadium or venue operator, event security contractor. Event work is seasonal and can be sold as a short engagement rather than a licence, which is unusually good for cash flow.

### G. Industrial logistics and supply operations

**Scenario.** A mine, port terminal, large warehouse campus or construction megaproject. Where is every vehicle and container, what's in stock, when does it run out, what's the best route given today's closures.

**Already carried.** The whole Epic 6 stack — real ERP change-data-capture via Debezium, depletion forecasting, hazard-aware routing — plus vehicle tracking via CV or GPS/IoT telemetry.

**Missing.** More ERP connectors (the CDC pattern generalises; SAP and Oracle are the ones that will be asked for).

**Buyer.** Operations or supply-chain director. Notably, this is the one vertical with a clean, countable ROI story — avoided stockouts and reduced vehicle idle time are numbers a customer can compute themselves, which shortens every argument about price.

### H. Defence and military C2 — *the original framing, the hardest market*

**Scenario.** The PRD's own: an operations centre fusing ISR feeds and generating courses of action for a commander.

**Already carried.** Everything. This is what was specified and what was built.

**Missing.** Nothing technical. Everything else: security clearances, facility accreditation, ITAR or equivalent export control, MIL-STD-2525 symbology, Link-16 or ATAK interoperability, a multi-year procurement cycle, and — most decisively — the kind of institutional credibility that a first-time vendor does not have.

**Verdict.** Do not lead with this. The civilian verticals above use the identical platform, pay faster, and build exactly the deployment track record that a defence conversation eventually demands. Going straight at defence as an unproven vendor is the most common way platforms like this run out of money.

---

## Ranked

| Vertical | Time to first revenue | Build gap | Regulatory drag | Typical deal | Verdict |
|---|---|---|---|---|---|
| **B. Critical infrastructure** | Medium | Small | Low–medium | Large, recurring | **Lead with this** |
| **A. Maritime** | Short | Small | Low | Medium, recurring | **Lead with this** |
| E. Drone / UAS ops | Short | Small | Low | Small, fast | Good first logo |
| G. Industrial logistics | Medium | Small | Low | Medium | Best ROI story |
| F. Smart city / events | Medium | Medium | **High** (privacy) | Medium | Only with a privacy posture |
| D. Border / wide-area | Long | Medium | Medium–high | Large | Later |
| C. Disaster response | Long | None | Medium | Grant-shaped | Credibility, not cash |
| H. Defence C2 | Very long | None | **Very high** | Very large | Not yet |

---

## The recommendation

**Pick critical infrastructure (B) and maritime (A). Build for both, because they need almost the same next three features.**

They share the work: ONVIF/radar connectors (`NW-019`), historical playback (`NW-018`), and multi-site or multi-AOI rollup. They share the pitch: fuse sensors you already own into one picture, with the AI running inside your own network. They are both civilian, so there is no export-control drag and no clearance requirement. And between them they cover both halves of the architecture — maritime proves the cloud profile, infrastructure proves the edge profile at remote sites.

Take drone-operations work (E) opportunistically as it appears; those are small, fast deals that generate references while the two main wedges go through longer cycles.

Revisit defence (H) once there are three civilian production deployments to point at. Everything built for A and B applies to H unchanged — that's the whole argument for sequencing it last rather than dropping it.

---

## What this document is not

It is not market research. No customer has been interviewed and no market size has been verified. Competitor pricing has now had one light pass (2026-10-16) — the results, and a blunt assessment of how trustworthy each source is, are in [19-commercialization.md](19-commercialization.md). Two figures there are primary-source; the rest are vendor blogs. What it *is* grounded in is the capability inventory at the top, which is real and verified, and the structural observation that those capabilities are domain-neutral.

Before committing to a wedge, the cheapest possible validation is roughly ten conversations with operators in verticals A and B — not to sell, but to find out whether the problem as described is the problem they actually have. That is a week of work and it is worth more than anything else in this document.

---
