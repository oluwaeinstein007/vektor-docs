# VEKTOR — Commercialization & Path to Revenue

[← Back to VEKTOR-PRD.md](../VEKTOR-PRD.md) · [Current System Status](16-system-status.md) · [Next Work](17-next-work.md) · [Use Cases](18-use-cases.md)

---

*Written 2026-10-16. Price figures in this document are **structural reasoning, not benchmarked numbers** — no competitor pricing was researched. See "What would change this document" at the end.*

## Where things actually stand

| | |
|---|---|
| Product | Feature-complete against its own PRD; twelve services, two apps, verified against real infrastructure |
| Deployed instances | **Zero** |
| Customers, pilots, design partners, LOIs | **Zero** |
| Revenue | **Zero** |
| Team | Far short of the sixteen people §18.1 models |
| Source code | All seven repos public on GitHub, **no licence file in any of them** |
| Hardware | No GPU, no Jetson — six roadmap tasks blocked on this |

The important asymmetry: the engineering is roughly 90% of the way to a sellable product and the business is at zero. That is an unusual and recoverable position — it is much easier to find a first customer for something that works than to build something that works for a customer you already signed. But it means **nothing in this document matters before [17-next-work.md](17-next-work.md)'s Tier 0 is done.** You cannot sell, pilot, or even credibly demo a system that has no built images and no deployed instance.

---

## Two decisions to make before pricing

### 1. The licence

All seven repos are public and unlicensed. Legally that means all rights reserved — nobody may use it. Practically it means the entire fusion engine, COA pipeline and RBAC layer are readable and cloneable by anyone, with no community upside in exchange. This is the worst of both positions and it is costing something every day it persists.

Three coherent options:

**Go private.** `gh repo edit --visibility private` on all seven, add a proprietary licence, sell closed-source. Simplest, and the default for a product with this security posture. Loses whatever credibility the public history provides — which, given the quality of the commit history and ADRs, is not nothing.

**Open-core.** Keep `vektor-proto`, `vektor-edge`'s tooling, the connector SDK (`NW-019`) and `vektor-docs` open under Apache-2.0; take `vektor-backend`'s differentiating services private or put them under a source-available licence (BSL, Elastic-style, or a fair-source variant with a time-delayed open release). The open half drives adoption and connector contributions; the closed half — `fusion-svc`'s EKF and correlation, `coa-svc`, `packages/auth` — is what people pay for. This fits the integrator story in [18-use-cases.md](18-use-cases.md) well.

**Fully open with paid everything-else.** Monetise support, hosting, certification and custom models. Viable, but it needs scale and a community before it pays, and it is the hardest of the three to run with a small team.

**Recommendation: open-core**, with the split above. It matches what's actually differentiated, it gives integrators something to build against without a contract, and it keeps the revenue-bearing code protected. But decide now — a licence applied after someone forks the repo protects nothing retroactively.

### 2. Who runs the infrastructure

§18.2 models $35,000–53,500/month of cloud and GPU spend, plus $850k–1.25M of on-prem CapEx. That is a cost model for a company operating a multi-tenant SaaS at scale. **At zero customers it is also a description of how to run out of money before the first invoice.**

The architecture offers a much better opening position: VEKTOR already runs fully self-hosted and fully offline, by design. The customer brings the hardware. Your marginal cost per deployment approaches zero, your gross margin is software margin, and — not incidentally — "it runs entirely inside your perimeter" is the exact sentence that wins the security conversation in verticals A and B.

**Sell self-hosted and edge first. Add hosted SaaS later, if customers ask for it** — and note from `NW-015` that hosted multi-tenancy isn't built, so "later" also means "after two to three weeks of real work you haven't done yet."

---

## How to price it

### The unit question

Pick the thing that scales with the customer's value and roughly with your cost. Four candidates:

| Unit | Scales with value? | Scales with cost? | Buyer understands it? |
|---|---|---|---|
| Per operator seat | Weakly | No | Yes, very |
| **Per connected sensor / stream** | **Yes** | **Yes** | **Yes** |
| Per site or edge node | Partly | Yes | Yes |
| Per event ingested | Yes | Yes | No — unpredictable, buyers hate it |

**Per connected sensor is the right primary unit.** A customer with 200 cameras gets more value than one with 10, and costs you more to serve (more inference, more storage). It's countable, it's forecastable for the buyer, and it grows naturally as they expand — which makes expansion revenue automatic rather than a renegotiation.

Seats as a secondary unit are worth including because buyers expect them and because they capture the organisational-adoption axis that sensor count misses. Pure consumption pricing is worth avoiding: operations buyers need a predictable annual number to get budget approved.

Either way, `NW-016` (usage metering) has to be built first. Nothing currently counts sensors, events, frames or COA generations.

### Packaging

Three tiers, mapping onto the three real deployment shapes the architecture already supports:

**Edge Appliance** — one edge node, offline-capable, fixed sensor count, single site. The disaster-response and remote-site shape. Annual licence per node, optionally bundled with hardware you resell at a margin. Lowest-friction entry point: one box, one price, no integration project.

**Operations Platform** — the full cloud-profile stack, self-hosted on the customer's infrastructure. Priced on sensor count with a seat component, annual commitment. Includes the dashboard, alerting, reporting, logistics, audit and RBAC. This is the main product for verticals A, B, D and G.

**Integrator / API** — access to the public REST and WebSocket API under a scoped service-account key, for partners embedding VEKTOR into their own tooling. Depends entirely on `NW-017`, which isn't built. Priced per key with rate tiers. Strategically this tier matters more than its revenue: G8 commits to three live third-party integrations by GA, and each integration makes the platform harder to displace.

On absolute numbers: the right band depends on what the vertical currently pays for the things VEKTOR replaces — the VMS, the AIS dashboard, the analyst hours — and that hasn't been researched. Structurally, pricing should land where a customer's annual VEKTOR cost is a visible fraction of the headcount or incident cost it displaces, and where a mid-size deployment is worth enough to justify the deployment effort a self-hosted install genuinely takes. Anchor on displaced cost, not on a cost-plus calculation from §18.2.

---

## Revenue beyond the licence

For a platform at this stage, services revenue is not a distraction from product revenue — it is how you survive long enough to have product revenue, and it is how you learn what to build.

**Deployment and integration.** A self-hosted install with real sensors is genuinely a project: connector work, model tuning, Keycloak federation, operator training. Charge for it. A fixed-fee deployment engagement per site is normal in this market and buyers expect it.

**Custom detection models.** The highest-margin line available, and the one closest to shipping. `cv-train-svc` already trains YOLOv8 and exports INT8 ONNX; `cv-inference-svc` already hot-swaps models with a verified upload endpoint. "We'll train a detector on your specific objects — your vehicle types, your equipment, your intrusion patterns — and deploy it without downtime" is a recurring, defensible, per-model service that competitors with fixed generic detectors cannot match. `NW-021` joins the existing pieces into a workflow.

**Connector development.** Every customer has one sensor nobody has written an adapter for. Build it as paid work, then fold it into the product. The architecture already supports this cleanly — adapters are isolated in `ingest-svc`.

**Support and SLA.** Standard tiers. Worth noting that an air-gapped edge deployment genuinely cannot be remotely supported, which makes on-site or scheduled-sync support a real, chargeable differentiator rather than an upsell.

**Training and certification.** [13-operator-manual.md](13-operator-manual.md) is already a per-role curriculum with seven incident runbooks. That is most of a training course that doesn't exist yet. Operations organisations pay for certified-operator programmes, and `NW-020`'s simulation mode would give it a live-fire component without touching real sensors.

---

## Sequencing

Five stages. Each has an exit gate — don't advance until it's met.

### Stage 1 — Make it demonstrable · ~4 weeks

[17-next-work.md](17-next-work.md) Tier 0 and the two one-hour items from Tier 1: `NW-011` (licence), `NW-005` (blue-force RBAC), `NW-003` (compose), `NW-001`/`NW-002` (images), `NW-004` (one deployed instance). Plus `NW-013` — measure the 60-second decision loop, because that number is the centre of every conversation that follows.

*Gate: a URL you can send a stranger, with a live pipeline behind it, and one real measured latency figure.*

### Stage 2 — Find out if you're right · ~4 weeks, overlapping

Ten to fifteen conversations with operators in verticals A and B from [18-use-cases.md](18-use-cases.md). Not selling — asking. What do they run today, what does it cost, what does it fail at, who signs. The demo from Stage 1 is the reason they take the call.

*Gate: three operators who say the problem as described is a problem they have, and at least one willing to be a design partner.*

This is also where a wrong assumption gets caught cheaply. If ten infrastructure operators say cross-camera identity isn't their pain, that is worth knowing before building `NW-019`.

### Stage 3 — One design partner, unpaid or near-free · ~3 months

A real deployment with real sensors at one site. The goal is not revenue — it is a reference, a case study with real numbers, and a brutally accurate list of everything that breaks in the field. Expect that list to be long and to reorder Tier 3 entirely.

Build during this stage only what the partner actually needs, which will likely be `NW-019` (their sensors) and `NW-018` (incident review).

*Gate: a production deployment the partner uses daily and will speak to a reference call about.*

### Stage 4 — First paid pilots · ~6 months

Two or three paid pilots at pilot pricing, from the Stage 2 pipeline plus whatever the Stage 3 reference opens. Now `NW-016` (metering) matters, because you need to know what you're billing for, and `NW-012`'s hardware matters, because pilots in the edge verticals need a real Jetson.

*Gate: a pilot converting to an annual contract at full price.*

### Stage 5 — Repeatable · ongoing

Standard pricing, a partner or integrator channel (`NW-017`), and the decision about whether hosted multi-tenancy (`NW-015`) is worth three weeks. Revisit defence (vertical H) here, with three civilian deployments to point at.

---

## The honest risks

**Deployment effort is the hidden cost.** A self-hosted install of a twelve-service distributed system with Kafka, three Postgres variants, Redis, Qdrant and Keycloak is not a weekend. The Helm charts exist, which helps enormously; the images don't, which is `NW-001`. If each deployment takes six weeks of engineering, the model doesn't scale past a handful of customers regardless of price. Measure it on the Stage 3 partner and treat the number as a core business metric, not an engineering detail.

**Solo-built platforms fail procurement on bus factor.** A buyer committing operational infrastructure asks who maintains it. The ADR set, the operator manual and the test discipline are a genuinely strong answer by the standards of an early company — lead with them.

**The LLM COA feature is the hardest thing to sell and the easiest to over-promise.** R-002 already identifies hallucination risk, and the human-in-the-loop requirement is a permanent anti-goal rather than a v1 limitation. Sell fusion and detection as the product; present COA generation as a decision aid that always requires approval. Over-claiming here is how a single bad recommendation ends a reference.

**Scale is unproven above a handful of entities.** NFR §8.3 commits to 100,000 active tracks and 500 concurrent operators; the system has been demonstrated with a few live entities and one load test at the Kafka layer. `NW-014` finds the real ceiling. Know it before a customer does.

**Zero market validation.** Every vertical ranking in [18-use-cases.md](18-use-cases.md), and every structural pricing recommendation here, is reasoning from capability rather than from evidence. Stage 2 exists to fix that and should not be skipped because Stage 1 was satisfying to finish.

---

## What would change this document

Four questions it can't currently answer, in order of how much they'd change the plan:

1. **What do vertical A and B operators pay today** for the VMS, AIS dashboards, analyst hours and incident tooling VEKTOR displaces? This sets the price bands that are currently left abstract.
2. **Who are the actual incumbents** in each wedge, and how are they packaged? Competitive pricing and packaging is researchable and hasn't been.
3. **How long does a real deployment take?** Unknowable before Stage 3, and it determines whether this is a product business or a services business.
4. **Is there a regulatory or compliance mandate** that funds the budget in verticals A and B? Mandated spend closes far faster than discretionary spend, and finding the mandate is often worth more than finding the feature.

---
