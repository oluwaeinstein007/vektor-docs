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

### What the market actually charges (researched 2026-10-16)

An earlier version of this section said the right price band "hasn't been
researched". It has now, lightly. **Read the source quality caveat at the
end before using any of these numbers in a conversation with a customer.**

**Video surveillance and analytics — the displaced cost in vertical B:**

| Product / model | Price |
|---|---|
| Cloud VSaaS, general | ~$10–50 per camera per month |
| Verkada | ~$200–1,800 per camera per year, plus ~$35–90 per camera per month |
| Avigilon Alta (subscription) | ~$179 per camera per year |
| Avigilon Enterprise (channel licence) | ~$335 MSRP per camera |
| Milestone (perpetual) | ~$70–250 per camera + $25–50 per year maintenance |
| Traditional enterprise perpetual | ~$200–300 per camera; maintenance 15–21% of licence per year |
| A 50-camera site, first year all-in | ~$45,000–150,000 |

**Maritime — the one published, primary-source figure found:** Sirius
Insight's maritime domain situational awareness service is listed on the UK
government's G-Cloud framework at **£100,000 per licence per year**, with a
one-month trial. That is a real government framework price rather than a
vendor estimate, and it is the single most useful anchor in this document.

**What funds the budget in vertical B:** NERC CIP-014 is the physical
security standard for substations, and in NERC's own response to a FERC
order utilities reported **$10–15 million** in substation security
spending. NERC also recorded roughly **1,700 physical security incidents**
to the E-ISAC in 2022, up 10.5% year on year. Violations reportedly carry
fines up to $1M per day per violation (vendor claim, not verified against
FERC). This is mandated, incident-driven spend, which is the shape that
closes fastest.

**The competitive set, as far as one pass found it.** Maritime: Sirius
Insight, Ultra Electronics (SAMS), e-BO e-Surveillance, eNav.my, Knogin.
Video/VMS: Genetec, Milestone, Verkada, Avigilon, Spot AI, Eagle Eye
Networks, Hikvision/Dahua. Notably, **none of the maritime vendors found
publishes pricing** beyond the one G-Cloud listing, which tells you
something about how that market sells.

### What this implies for VEKTOR's price

Per-camera VMS pricing sets a floor that is lower than instinct suggests. A
200-camera infrastructure site is paying on the order of $36,000–100,000 a
year for Avigilon-or-similar *surveillance alone*, before analytics. VEKTOR
is not competing with that line item — it sits above it, replacing the
analyst hours and the second and third screens — but a per-sensor price
that lands far outside that band will be compared to it anyway, fairly or
not.

The maritime £100,000/year figure is the more encouraging anchor, and the
more representative one: it prices a *platform* for an agency, not a
per-device licence, which is the shape VEKTOR's Operations Platform tier
should take.

So: anchor the Operations Platform tier against the maritime figure and the
displaced-analyst-headcount argument, not against per-camera VMS. Use
per-sensor pricing to scale *within* an account rather than to set the
entry price. And in vertical B, lead with the compliance mandate — $10–15M
substation security programmes and $1M/day exposure make a six-figure
platform an easy line item, where the same number is a hard sell as
discretionary spend.

**Source quality, stated plainly.** Only two figures here are
primary-source: the Sirius G-Cloud listing and the NERC/FERC substation
numbers. Everything in the per-camera table comes from vendor blogs,
reseller comparison pages or AI-generated summaries, several of which
disagree with each other, and none of which is specific to critical
infrastructure. They are order-of-magnitude orientation, not benchmarks.
**Before pricing anything, get three real quotes** — Genetec or Milestone
for the VMS baseline, and a maritime vendor for the platform comparison —
and look at public tender notices (TED in Europe, SAM.gov in the US), where
real procurement prices are published and nothing above has to be taken on
trust.

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

1. ~~**What do vertical A and B operators pay today?**~~ — **partly answered
   2026-10-16**, see "What the market actually charges" above. Still open:
   nothing found prices the *analyst hours* VEKTOR displaces, which is the
   larger half of the ROI argument and the one a customer can compute for
   themselves.
2. ~~**Who are the actual incumbents?**~~ — **named above**, though only one
   of them publishes a price. The remaining gap is how the maritime vendors
   package and what a real port authority contract contains; public tender
   notices (TED, SAM.gov) are where that lives and were not searched.
3. **How long does a real deployment take?** Unknowable before Stage 3, and it determines whether this is a product business or a services business.
4. **Is there a regulatory or compliance mandate** that funds the budget in verticals A and B? Mandated spend closes far faster than discretionary spend, and finding the mandate is often worth more than finding the feature.

---

---

## Sources

Pricing and market figures in "What the market actually charges" come from:

- [Sirius Insight — Maritime Domain Situational Awareness, UK G-Cloud](https://www.applytosupply.digitalmarketplace.service.gov.uk/g-cloud/services/981021616158464) — the £100,000/licence/year figure. **Primary source.**
- [FERC — Evaluation of the Physical Security Reliability Standard](https://www.ferc.gov/news-events/news/presentation-evaluation-physical-security-reliability-standard-and-physical) and [NERC CIP physical security overview](https://securithings.com/blog/nerc-cip-101-what-does-it-mean-for-physical-security) — the $10–15M substation spend and the E-ISAC incident count. **Primary for the FERC/NERC figures.**
- [Avigilon pricing 2026](https://www.spot.ai/blog/avigilon-pricing-2026) and [Eagle Eye Networks pricing 2026](https://www.spot.ai/blog/eagle-eye-networks-pricing-2026) — competitor-published comparisons; treat as directional.
- [Video surveillance software companies](https://www.forasoft.com/blog/article/best-video-surveillance-software-companies), [VMS architecture 2026](https://www.forasoft.com/blog/article/video-management-systems-vms-2026-architecture), [video surveillance management systems](https://www.forasoft.com/blog/article/video-surveillance-management-systems) — vendor blog; the per-camera and fleet-size ranges.
- [Enterprise VMS licence and maintenance ranges](https://checkthat.ai/ai-answers/eb321771-e2aa-435a-9f87-7496b4b5081b) — AI-generated summary; the weakest source here, included only because it agrees with the others.
- Maritime vendor landscape: [Ultra SAMS datasheet](https://www.ultra-hyperspike.com/media/2333/sams-maritime-datasheet-042821_web.pdf), [e-BO maritime surveillance](https://ebo-enterprises.com/navigate/maritime-surveillance/), [Knogin maritime intelligence](https://knogin.com/developers/maritime-intelligence), [eNav.my](https://edmglobaltech.odoo.com/enav).

**Two of these are primary sources and the rest are not.** The per-camera
table should be replaced with real quotes before it informs a pricing
decision.
