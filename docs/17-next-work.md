# VEKTOR — Next Work: Tasks & Features

[← Back to VEKTOR-PRD.md](../VEKTOR-PRD.md) · [Current System Status](16-system-status.md) · [Use Cases](18-use-cases.md) · [Commercialization](19-commercialization.md)

---

*Written 2026-10-16, against the repos as they stood that day. Like [16-system-status.md](16-system-status.md), this is a dated snapshot — trust the date, not the file.*

## Why this doc exists

[09-roadmap.md](09-roadmap.md) describes Phases 0–7, and all of them are closed. That roadmap was written to answer "what do we build?" — it never had to answer "what do we do with it once it's built?" This doc is the backlog for the period *after* the roadmap ran out.

It is deliberately ordered by **what unblocks what**, not by what's interesting. Each task carries an `NW-xxx` ID so it can be referenced from commit messages and the other docs.

---

## The verdict in one paragraph

VEKTOR is **feature-complete against its own specification and not deployable by anyone**. Twelve backend services, two frontend apps, and a real verified pipeline exist — ingest through CV through EKF fusion through LLM-generated COAs through alerting, logistics and reporting, every phase tested against real infrastructure rather than mocks. But there is no built image for nine of the twelve services, no CI job that publishes an image anywhere, no cloud-profile compose file, and no deployed instance. The gap between this codebase and a running system someone else can log into is small in engineering terms and total in commercial terms. **Tier 0 below is that gap, and it is the only work that matters until it's closed.**

---

## Status as of 2026-10-16 (same day)

Work started against this list immediately. What has actually landed, all
verified against real infrastructure rather than asserted:

| Task | State | Evidence |
|---|---|---|
| `NW-001` Dockerfiles | **done** | All 12 services have one (was 3), and **all 12 images build and run** — `reporting-svc` needed three attempts, failing twice on transient network (an apt mirror `Err:78 libflac12`, then an IPv6 route to auth.docker.io), never on its Dockerfile. Every image was then verified to load its full module graph, not merely to tag: each exits on its own config guard (`DATABASE_URL is required`), which is what proves the cross-repo `link:@vektor/proto` resolves inside the image. `onnxruntime-node` and `better-sqlite3` load with working native bindings; `reporting-svc` carries Chromium 155 at the path its `CHROME_EXECUTABLE_PATH` expects. `geospatial-svc` was smoke-tested end to end: real container → compose Postgres → real Keycloak JWT → 401 unauthenticated, 200 with an Analyst token |
| `NW-002` publish CI | **written, unrun** | `.github/workflows/release-images.yml` — 12-service matrix, amd64+arm64, GHCR. Cannot be verified without pushing to the real registry |
| `NW-003` cloud compose | **done** | `compose/docker-compose.dev.yml` + `scripts/dev-stack.sh`. 7 core containers healthy; all 9 migrations apply to an empty database; fusion-svc 41/41 and reporting-svc 11/11 run against it |
| `NW-004` deployed instance | **prepared, not deployed** | Per your call: everything builds and runs locally; the deploy step is yours |
| `NW-005` blue-force RBAC | **done** | 7 routes gated, frontend switched to authenticated fetch, 4 new tests, fusion-svc 41/41 green |
| `NW-006` CI test gaps | **partly done** | `packages/auth` + `reporting-svc` added after verifying what they really need. `coa-svc` and `logistics-svc` are still out of CI, but both have now been **run for real** against the full stack — coa-svc 42/42, logistics-svc 14/14 — so what they need is known rather than guessed |
| `NW-007` GGUF fixture | **was already fixed** | See correction below |
| `NW-008` coverage | **done** | `test:coverage` on 17 packages. First measured number: `@vektor/auth` 96.61% lines / 81.58% branches |
| `NW-009` dep audit | **done** | 4 critical/48 high → **0 critical/2 high**. 13 `pnpm.overrides`, nodemailer 9→10 (verified by a real SMTP send), new CI `audit` job that fails on high/critical |
| `NW-013` decision-loop latency | **done** | New `@vektor/latency-harness`. Sensor→operator **p95 1155 ms tuned / 1965 ms default**, 25/25 received. G1 uses ~2–3% of its 60 s budget; NFR 8.1's 500 ms fusion target is **not** met |
| `NW-014` map perf | **done** | New `apps/web/scripts/map-perf-bench.mjs` — real MapLibre in real headless Chrome, 100→5000 entities |
| `NW-010` SEC-005 LUKS | **done by a peer session** | Delivered concurrently by session `vektor-2e` — 9 files in `vektor-edge`, verified against real cryptsetup 2.7.0/LUKS2. Found two genuine bugs on the way: k3s was bootstrapping without `--secrets-encryption` and onto a plaintext data dir |
| `NW-018` historical playback | **done by a peer session** | Delivered by `vektor-2e`. New `entity_history` table (RANGE-partitioned by day), a Kafka projector, 4 `analyst+`-gated routes in `geospatial-svc`, and a time-slider + track layer in `apps/web`. 19 backend + 18 frontend tests; geospatial-svc 30/30 against real PostGIS |
| `NW-011` licence | **done** | Open-core: Apache-2.0 on `vektor-proto`/`-edge`/`-infra`/`-docs`, BSL 1.1 (→ Apache-2.0 on 2030-10-16) on `-backend`/`-web`/`-ml`; 27 `package.json` files moved off `UNLICENSED` |
| `NW-020` scenario player | **done by a peer session** | Delivered by `vektor-2e` as `@vektor/scenario`. Deterministic: same scenario and seed give byte-identical `--dry-run` output, and jitter is keyed per (seed, entity) so adding an entity cannot silently change every other track. 22 tests, verified driving a live fusion-svc. **Caveat recorded in its README:** above `--speed 1` the EKF multiplies derived velocity by playback speed (positions advance at scenario rate, timestamps at wall-clock rate) — a 13-knot vessel reads 9,669 km/h at 30×. Use `--speed 1` for anything calibrated |
| `NW-015` multi-tenancy | **groundwork done, decision still open** | `organisations` + `org_id` on all 15 tenant-scoped tables (including the partitioned `entity_history`), `withOrgScope()`, RLS policies, and `org_id` on the verified JWT principal. **Deliberately does not decide whether VEKTOR should be multi-tenant** — see the note below. A no-op for single-tenant. packages/db 12/12 through a real non-owner role with two organisations' rows in the same tables |
| `NW-016` usage metering | **done** | New `@vektor/metering`. Six billable axes, emitted fire-and-forget (batched, bounded, lossy under pressure by design) and rolled up per (organisation, metric, hour). The rollup is **exactly-once**: the consumer's Kafka offset is committed in the same transaction as the increment, so a replayed batch cannot double-count a number that ends up on an invoice. 14/14 against real Postgres |
| `NW-017` integrator API keys | **done** | `vk_<prefix>_<secret>` keys, org-scoped, with scopes, per-key rate limits and timestamp revocation. `requireRole()` is unchanged and now accepts either credential, so every already-gated route works with a key untouched. Not OAuth2-against-Keycloak, deliberately — that would leave the edge profile, which runs with no Keycloak, unable to authenticate an integration. packages/auth 21/21 |
| `NW-019` connector SDK | **SDK + 3 reference connectors done; existing adapters not ported** | New `@vektor/connector-sdk`. Three connectors of three deliberately different shapes — `modbus-tcp` polls, `lorawan-webhook` listens, `onvif-discovery` resolves — because an interface that only fitted pollers would be a poller framework wearing a connector's name. Modbus and LoRaWAN are verified against a real TCP server speaking real MBAP framing and a real HTTP server; **ONVIF has never seen a real camera** and says so. 18/18. The SDK also fixes a latent defect it was built from: `aisAdapter` never reconnects, so a dropped AIS feed stops silently with ingest-svc still healthy |

### The multi-tenancy decision is still open, and that is deliberate

`NW-015` shipped the *groundwork* and pointedly not the *decision*. The schema
now carries an organisation key everywhere, defaulted so a single-tenant
deployment behaves exactly as it did before. What that buys is optionality:
the expensive, irreversible half of multi-tenancy is retrofitting a tenant key
onto fifteen tables that already hold production data, and that is now done
while it was still free.

Someone still has to choose. The case each way, briefly:

**Stay single-tenant.** [18-use-cases.md](18-use-cases.md)'s two
strongest-ranked verticals — maritime domain awareness and critical
infrastructure — are both shaped like self-hosted installs and edge
appliances, not like SaaS tenants. "No data leaves your boundary, ever"
is the sentence that ends the security objection in both, and a hosted
multi-tenant offering weakens it. Single-tenant also means the RLS work
below stays a defence-in-depth backstop rather than the primary control.

**Go multi-tenant.** It is the only way to serve small customers whose whole
annual contract is less than the cost of standing up an install for them,
and [19-commercialization.md](19-commercialization.md)'s Stage 5 assumes a
repeatable motion that per-customer deployments do not give you.

**What would decide it:** whether the first three paying customers want to run
it themselves. That is knowable from Stage 2 of
[19-commercialization.md](19-commercialization.md)'s sequencing and not before,
so the honest answer today is "not yet, and nothing is blocked by not knowing".

One deployment note that applies either way: **the RLS policies are inert
today.** They are scoped `TO vektor_app`, Postgres never applies RLS to a
table's owner or to a superuser, and every `DATABASE_URL` in this repo still
connects as `postgres`. This is the same gap SEC-003 left on `audit_log` in
migration 0007. Closing it means switching each service's connection identity
to `vektor_app` as part of an infra/secrets rollout (INFRA-010) — it is not a
migration, and no migration can do it.

### Bugs found and fixed while doing the above

None of these were in any issue list; all were found by actually running
things, and all are fixed:

1. **Every Dockerfile and the ARM64 CI job were broken from a clean
   checkout.** `vektor-proto/src/generated/` is gitignored and `src/llm.ts`
   re-exports from it, but no Dockerfile and no workflow ran `protoc`. They
   only ever worked on a machine where a developer had already run codegen
   locally and it got swept into the build context. Reproduced
   (`TS2307: Cannot find module './generated/llm.js'`), fixed in all 12
   Dockerfiles, and `vektor-proto/src/generated` added to `.dockerignore` so
   a local build can never again differ from a clean one.
2. **The shared `.dockerignore` lived only on one machine.** It sat in
   `VEKTOR/`, which is not a git repository, so CI built with no ignore file
   at all. A tracked copy is now `vektor-backend/.dockerignore`.
3. **Migration 0007 grants to a role nothing creates.** `drizzle-kit migrate`
   against a genuinely fresh Postgres failed with
   `role "vektor_app" does not exist` — only `rls.test.ts` ever created it.
4. **`ingest-svc` and `reporting-svc` both defaulted to port 3011.** A
   same-machine run of both collided. `ingest-svc` moved to 3016, which is
   what `vektor-web`'s `api-client.ts` had assumed all along.
5. **`vektor-edge`'s compose pointed `ALERT_SVC_URL` at 3011** — that is
   reporting-svc, not alert-svc (3013). The identical bug `edge-sync-svc`
   had already fixed in its own code.
6. **`ci.yml`'s maildev was unpinned.** MailDev moved its REST API from
   `/email` to `/api/email` between 2.x and 3.x; every mail-delivery
   assertion in the repo uses `/api/email`. Pinned to 3.0.0 in both CI and
   compose.
7. **`packages/auth` was excluded from CI for a reason that isn't true.**
   The comment said its tests need a live Keycloak realm; they use an
   in-process keypair and pass 5/5 with no infrastructure at all.
8. **Keycloak rejected every imported realm user** with
   `"Account is not fully set up"`. Two stacked causes: a realm import
   replaces a user's realm-role set, so `default-roles-vektor` has to be
   listed explicitly; and the users had `lastName: "(dev)"`, which the
   default declarative user profile's `person-name-prohibited-characters`
   validator rejects — triggering `VERIFY_PROFILE` *dynamically*, so the
   admin API reports `requiredActions: []` while every password grant keeps
   failing. A parenthesis in a name field.
9. **Migration 0007 still broke a plain `drizzle-kit migrate`** even after
   the compose init script "fixed" it — the script only helps people using
   this repo's own dev stack. Raised by a peer session, and correct. 0007
   now creates the role itself in a `DO` block. Verified against a virgin
   cluster (applies clean) *and* against the already-migrated dev database
   (no re-run, no hash mismatch — drizzle keys off the journal timestamp).
10. **`@grpc/grpc-js` must be bumped in `vektor-proto` and `vektor-backend`
   together.** Updating one desyncs the two copies, and `coa-svc` fails with
   `TS2322 ... not assignable to UntypedHandleCall` — the cross-repo gRPC
   class-identity pitfall, reached by a new route.
11. **The `maildev` healthcheck I wrote was for 2.x.** 3.x serves health at
   `/api/healthz`, so the container sat permanently "unhealthy" while
   working perfectly.

12. **`pgrouting/pgrouting:16-3.4-3.6` has never existed on Docker Hub**, so
   `pnpm stack:up:full` died with `manifest unknown` and logistics-svc's
   pgRouting tests had no instance to run against. Found by a peer session
   reviewing the compose file I'd just written. Corrected to
   `16-3.4-3.6.1`.
13. **An unbounded `>=` pnpm override crossed a major and broke a client.**
   `undici: ">=7.29.1"` resolved to 8.12.2; undici 8 made
   `handler.onRequestStart` mandatory where 7 feature-checked it, so every
   `@qdrant/js-client-rest` request threw `InvalidArgumentError` before
   leaving the process. `js-yaml` had silently crossed into 5.x by the same
   mechanism. All 13 overrides are now bounded to the major that contains
   the fix, and the audit result is unchanged at 0 critical / 2 high.
   **General lesson: a bare `>=X` override is a landmine whenever the
   package's current major is above X.**
14. **`fusion-svc`'s `pipeline.integration.test.ts` is not self-cleaning.**
   It leaves fused entities in the shared `entities` table, and a later run
   fails with a mismatched `entity_id` because cross-sensor correlation
   matches a leftover track instead of the one the test just created. It
   passes 41/41 against an empty table and fails against a dirty one, which
   makes it look like a flake and is really a missing teardown — another
   instance of the shared-table race this project has already hit twice.

### `NW-022` — Decide whether an alert should auto-trigger a COA · product decision, then ~3 days

Found while verifying NW-013 (by `vektor-2e`). **G1's "sensor-to-decision
in under 60 seconds" has a human in the middle of it today.**
`POST /api/v1/coa/generate` is the only way to start COA generation, it is
operator-driven, and nothing in the repo calls it — `coa-svc`'s own route
comment anticipated `alert-svc` as the automatic caller "once it exists",
and `alert-svc` now exists and does not. §9.7's data-flow diagram
("coa-svc triggered by alert or operator request") describes a leg that was
never built.

Wiring `alert-svc` → `coa-svc` is a few days of work. **Whether to is not
an engineering question.** The product's first permanent anti-goal
(00-vision.md §3.2) is that VEKTOR never autonomously executes a response
and every action requires human approval. Auto-*generating* a recommendation
is not auto-executing one, and a COA sitting ready when the commander opens
the alert is exactly the decision-loop compression G1 promises. But it also
means the system produces unprompted recommendations for action, which
deserves an explicit decision and an ADR rather than arriving as a
side-effect of a latency task.

Until it is decided, G1 should be stated as two numbers — the machine leg
(measured, [15-perf-results.md §26a](15-perf-results.md)) and the operator
leg (unmeasurable, and not the system's to claim).

### The altitude-provenance defect, and why it took three fixes (2026-10-18)

Recorded because the shape generalises well beyond this one field.

fusion-svc's correlation gate used horizontal distance only, so an aircraft
at 3,000m overflying a vessel merged into the vessel's track at an averaged
1,508m and one real contact vanished — silently, with the surviving entity
looking normal. Found by driving the pipeline with a scenario (NW-020), not
by reading the code.

Fixing it took three passes, and **each fix was correct in isolation while
the bad value kept flowing to the next consumer**:

1. The gate ignored altitude. Added a vertical gate, checked against the
   EKF's *predicted* altitude so a climbing aircraft still correlates with
   itself, with `ewrf` exempt because RF geolocation has no usable altitude.
2. The ADS-B mapper coerced a nullable `altitude_ft` to `0`, so "not
   reported" was indistinguishable from "at sea level" and an altitude-less
   aircraft still merged with ships. Added `alt_known`.
3. The EKF was *still* being handed that placeholder `0` as a measurement
   by the EW/RF mapper. Four RF fixes dragged a known-good 3,000m aircraft
   to **−247m**, below sea level. Made `alt_known: false` suppress the
   altitude update, not just the gate.

**This is a provenance problem, not three bugs.** A placeholder substituted
for a missing measurement will be read as real by whatever consumes it
next, and fixing the consumer in front of you does not stop it reaching the
others.

~~Worth checking for the same pattern elsewhere~~ — **that sweep has been
done.** Thirteen `?? 0` hits across the backend and web source; twelve
benign (counters, config defaults, SQL aggregates, and the one mapper where
the placeholder is correctly paired with `alt_known`), and **one real, in
`packages/scenario` — the tool that found the original defect**. A scenario
aircraft authored without an altitude emitted `altitude_ft: 0,
on_ground: true`, which after these fixes means a *known* sea-level
altitude, so it would merge with any vessel beneath it. Full result in
[15-perf-results.md §26c.4](15-perf-results.md).

The part worth remembering: that instance was written by the same engineer
who wrote the report naming the pattern, in the same change. The defect is
not visible from inside the code you are writing, which is the argument for
sweeping for it mechanically and periodically rather than treating it as
fixed.

All four end-to-end cases now pass, verified independently by a second
session against an isolated stack — see
[15-perf-results.md §26c](15-perf-results.md), which also records the
non-obvious way to reset fusion-svc between runs (`FLUSHALL`, restart,
`TRUNCATE` — a truncate alone leaves in-memory tracks and Redis-buffered
observations that silently repopulate the table).

Two related defects fixed alongside, both found by the same verification:
`/healthz` returned 200 while the pipeline failed every cycle — 117
consecutive failures behind a green probe, which on Kubernetes means no
restart and no alert — and Kafka consumer group ids were not namespaced by
`VEKTOR_ENV`, so two instances on one broker joined the same group and each
silently received a subset of the other's topics.

### NW-018's four silent-failure bugs (found by `vektor-2e`, recorded here because each one looked like success)

Worth keeping because none of them raised an error — each produced
plausible-looking wrong output:

1. **`ST_SimplifyPreserveTopology` silently drops the M dimension**
   (PostGIS 3.4.3; `ST_Zmflag` 1 → 0). Track decimation encodes each
   point's timestamp in M so surviving vertices can be joined back to their
   rows. With topology preservation every `ST_M()` returned NULL, the join
   matched nothing, and an entity with 1,440 points returned an **empty
   track**. `ST_Simplify` preserves M — topology preservation is a polygon
   concern and a track is a LineString.
2. **`ST_Length(ST_Envelope(geom))` is always 0.** An envelope is a
   polygon, and `ST_Length` of a polygon is 0, not its perimeter. A derived
   decimation tolerance built on it silently collapsed to its floor every
   time.
3. **Spatial DBSCAN over raw track points cannot find dwells** — and fails
   in a way that looks reasonable. A moving entity is spatially dense along
   its own path (at 14 km/h sampled every 10s, consecutive points are ~39 m
   apart), so at any useful `eps` DBSCAN chains an entire journey into one
   cluster whose centroid sits mid-transit, at a place the entity never
   stopped. Measured: 1 cluster, 1 "visit", a 14,390-second dwell, centroid
   near neither berth. The fix is stay-point detection — derive speed from
   consecutive positions via `::geography`, flag stopped, gaps-and-islands,
   then cluster the *run centroids* — which correctly returns 2 visits of
   ~7,180s.
4. **drizzle-orm's raw `sql` tag cannot bind a JS `Date`.** It passes
   parameters to the `postgres` driver untouched and you get
   `ERR_INVALID_ARG_TYPE ... Received an instance of Date` at bind time —
   the same class as the array-parameter trap already documented in
   `tests/app.test.ts`. Timestamps must go in as ISO strings with an
   explicit `::timestamptz` cast. This was in production code, not tests.

A design note worth carrying: the default decimation tolerance is a
**display-resolution rule** (drop detail finer than one pixel at full-track
zoom), not budget-derived. Budget-derived was tried and measured to be
unworkable — sweeping the divisor returned 4 points at every tolerance from
108 m down to 4.3 m, then 264 at 1.08 m, because the cliff is GPS jitter
crossing the threshold. Output size is governed by sensor noise, not by the
caller's budget, so a geometrically simple track legitimately returns far
fewer points than `max_points`; `total_points`/`decimated` tell the operator
what was dropped.

### Corrections to this document

- **`NW-007` was already resolved before this document was written.** The
  469MB GGUF is committed as a proper 134-byte Git LFS pointer,
  `.gitattributes` carries `*.gguf filter=lfs`, git-lfs 3.6.1 is installed
  and `git lfs fsck` is clean against the pushed remote. Both this file and
  [16-system-status.md](16-system-status.md) listed it as an open gap; it
  isn't one. `coa-svc`'s edge test also self-skips when the fixture is
  absent, so it never blocked CI in the first place.
- `NW-012`'s hardware purchase and `NW-004`'s deploy are the two items that
  cannot be done from a development machine at all. Everything else in
  Tiers 0–2 is now either done or unblocked.

---

## Tier 0 — Nothing ships until these are done

Estimates assume one engineer already familiar with the codebase. They are rough.

### `NW-001` — Dockerfiles for the nine services that have none · ~3–4 days

Three services have a Dockerfile (`cv-inference-svc`, `coa-svc`, `edge-sync-svc`) — all three written during Phase 6 for the edge profile. **Nine do not**: `alert-svc`, `audit-svc`, `fusion-svc`, `geospatial-svc`, `ingest-svc`, `logistics-svc`, `map-tile-server`, `mavlink-bridge`, `reporting-svc`.

This is not cosmetic. Eleven services ship a Helm chart under `services/*/deploy/helm/`, and every chart's `values.yaml` points `image.repository` at a `ghcr.io/oluwaeinstein007/vektor-*` tag. **Nothing builds those images and nothing publishes them.** `fusion-svc`'s own `values.yaml` header comment says so in as many words. Every ArgoCD `Application` in `vektor-infra` therefore references a chart that references an image that does not exist.

The three existing Dockerfiles already solved the hard part — they build from the `VEKTOR/` parent directory as context so the cross-repo `link:../../../vektor-proto` dependency resolves. Copy that pattern. Note that `reporting-svc` needs a Chrome binary in the image (it uses `puppeteer-core` against a system Chrome, deliberately not a bundled download) and `logistics-svc`/`edge-sync-svc` have native modules that need build toolchains present at install time.

`mavlink-bridge` has neither a Dockerfile nor a Helm chart — it's the one service that postdates the Phase 6 deployment work entirely.

### `NW-002` — A CI job that actually publishes images · ~1–2 days

`.github/workflows/edge-arm64.yml` builds three ARM64 images on every relevant PR and then throws them away: `push: false`, line 61. `ci.yml` builds, lints and tests, and produces no artifact at all.

Needed: a release workflow that logs into GHCR, builds every service for `linux/amd64` and `linux/arm64`, tags by git SHA and by semver, and pushes. Then set each chart's `appVersion` from the same release so `image.tag` resolves to something real. Until this exists, `NW-001` produces files nobody can consume.

### `NW-003` — A cloud-profile `docker-compose.yml` · ~2 days

The only compose file in all seven repos is `vektor-edge/compose/docker-compose.edge.yml`, which is edge-profile only. [16-system-status.md](16-system-status.md) tells a newcomer to bring up Postgres, Kafka, Redis and Qdrant by hand from individual service READMEs. NFR §8.6 commits to `pnpm dev` standing up the full stack in under two minutes.

This is the highest-leverage two days in the whole document. It is what turns "clone and read" into "clone and run", and every demo, every design partner, every new contributor and every item in Tier 2 depends on it. Include the dev auth bridge (`pnpm --filter @vektor/auth dev:auth-bridge`) so `fusion-svc`'s mandatory `KEYCLOAK_JWKS_URI` is satisfied out of the box, and seed enough entities that the dashboard renders something on first load rather than an empty map.

### `NW-004` — One deployed instance, at one URL · ~1 week

A single cluster (k3s on one rented box is enough) running the real images from `NW-002` via the existing Helm charts and ArgoCD manifests, with real TLS and real Keycloak. Not a product launch — an address you can send someone.

Everything downstream of this document assumes it exists. You cannot run a pilot, a security review, a performance benchmark or a sales demo against a laptop.

---

## Tier 1 — Nothing sells until these are done

These are the things a competent buyer-side security or procurement review will find. Each one is cheap now and expensive at the wrong moment.

### `NW-005` — RBAC gate on `fusion-svc/src/blueforce/routes.ts` · ~half a day

Every other route surface in the system went through SEC-003 and carries `requireRole()`. `blueforce/routes.ts` does not — confirmed by grep, and already flagged in [16-system-status.md](16-system-status.md) as "flagged, not fixed".

These are the **blue-force asset and no-strike-zone endpoints**. They are the safety-critical control surface of the entire platform: the data that stops a COA from recommending action against a friendly or protected position. An unauthenticated caller can currently read the full friendly-asset picture and modify no-strike zones. Of every open item in this document this is the one that is both smallest and worst.

### `NW-006` — Close the CI test-coverage gaps · ~3 days

`ci.yml`'s test job runs eleven workspaces and documents, honestly, exactly which ones it excludes and why: `coa-svc` (needs Qdrant + BGE-M3 + a stand-in gRPC LLM server + a 469MB GGUF fixture that isn't committed), `logistics-svc` (needs three Postgres instances plus a Debezium/Kafka Connect stack), `reporting-svc` (needs headless Chrome), `packages/auth` (needs a Keycloak realm), `packages/qdrant` and `packages/load-test` (never wired in).

That is six workspaces — including two of the most business-differentiating services in the system — whose tests have never run in CI. `reporting-svc` and `packages/auth` are easy wins (`browser-actions/setup-chrome`, and a Keycloak service container with an imported realm JSON). `logistics-svc` needs a compose-based job but no new infrastructure thinking — `NW-003` will have already written most of it. `coa-svc` needs the fixture question answered first.

### `NW-007` — Make the GGUF fixture committable · ~1 day

`coa-svc/tests/fixtures/qwen2.5-0.5b-instruct-q4_k_m.gguf` is 469MB against GitHub's 100MB hard limit. It has been flagged as unresolved since Phase 6. Either Git LFS, or — better — fetch it in CI from a pinned HuggingFace URL with a checksum assertion and keep it out of the repo entirely. Blocks the `coa-svc` half of `NW-006`.

### `NW-008` — Measure test coverage at all · ~1 day

NFR §8.6 commits to ≥80% unit coverage. **Nothing in any repo measures coverage** — no `c8`, no `nyc`, no coverage flag anywhere. Every service's test script is a bare `node --test dist/tests/*.test.js`. Node 22 has `--experimental-test-coverage` built in; wire it, publish the number, and then decide whether 80% is the right target. Today the NFR is unfalsifiable, which is worse than missing it.

Related: coverage is very unevenly distributed. `coa-svc` and `fusion-svc` have 10 test files each; `geospatial-svc`, `map-tile-server` and `mavlink-bridge` have one apiece.

### `NW-009` — Re-run the dependency audit and burn down what's left · ~2 days

[14-sec004-security-audit.md](14-sec004-security-audit.md)'s counts (1 critical, 11 high, 7 moderate) are from **2026-08-29** — seven weeks stale as of this writing. SEC-004 also left `vektor-proto`, `vektor-edge` and `vektor-ml`'s Python-side scans inconclusive for environmental reasons, documented as open rather than clean. Re-run all of it, and add `pnpm audit` / `pip-audit` as a CI job so the number is never seven weeks old again.

### `NW-010` — SEC-005: LUKS full-disk encryption on edge nodes · ~2 days

The one roadmap task never attempted. It matters disproportionately for the commercial story: an edge node is a physical box left in a field, a vehicle, or a disaster zone, and "what happens when it's captured or stolen" is a question every serious buyer in every vertical asks. `vektor-edge`'s bootstrap scripts are the right home.

### `NW-011` — Put a licence on the repositories · ~1 hour

All seven repos are **public on GitHub** (verified 2026-10-16) and **not one contains a LICENSE file**. `vektor-backend/package.json` says `"license": "UNLICENSED"`, which has no legal effect on its own.

Public with no licence means all rights are reserved by default, so nobody may legally use it — while the entire source, including the fusion engine and the COA pipeline, is readable and cloneable by anyone. That is the worst of both: no community benefit, no legal protection in practice, and a licensing question a prospective customer's legal team will raise on day one. See [19-commercialization.md](19-commercialization.md) for the open-core options; the decision is a business one, but *some* decision needs making this week, not after someone has forked it.

---

## Tier 2 — Claims nobody has measured

§3.1 and §8.1 make twenty-odd numeric commitments. Exactly one has been measured end to end: PERF-001's 9,889 events/sec sustained, documented with its own caveats in [15-perf-results.md](15-perf-results.md). The rest are either hardware-blocked or simply never attempted.

### `NW-012` — Buy the hardware loop · ~$3–6k, then ~1 week

PERF-002 (<200ms/frame GPU inference), PERF-003 (<30s COA on Jetson), PERF-004 (<10s COA on cloud GPU), ML-002's GPU path, ML-007's vLLM engine, and EDGE-002/003's Jetson throughput are all blocked on the same thing: **no GPU and no Jetson has ever existed in any environment this was built in.** Six roadmap tasks, one purchase.

A Jetson AGX Orin dev kit plus an hour or two of rented H100 time resolves all six. This is the single highest-value spend available and it is small. Until it happens, the edge story — the thing the product's own name is built around — rests entirely on an emulated QEMU check that `onnxruntime-node`'s ARM64 binding loads and produces correctly-shaped output.

### `NW-013` — Measure the headline claim: sensor → decision in under 60 seconds · ~3 days

G1 is the product's central promise and **nothing measures it**. Every component has been timed in isolation; the composed path — a frame arriving at `ingest-svc`, through CV inference, through fusion, through COA generation, to a rendered recommendation — has never been instrumented end to end.

Emit an OpenTelemetry trace spanning the whole path (the observability stack is already specified in INFRA-009) and publish the real distribution. If it's under 60s, that number is the most valuable sentence in any sales conversation. If it isn't, you need to know before a customer tells you.

### `NW-014` — Map performance at 1,000+ tracks · ~2 days

NFR §8.1 commits to 60fps on the WebGL map with 1,000+ active tracks. The dashboard has only ever been demonstrated with a handful of real entities. ~~MapLibre will need symbol clustering or a switch to a custom layer well before 1,000~~ — **measured, and that prediction was wrong by about an order of magnitude.** On a 2016 Intel HD 520 integrated GPU the frame-rate curve is flat from 100 to 5,000 entities; the knee sits between 5,000 and 10,000, and no clustering is needed at 1,000. See [15-perf-results.md §26b](15-perf-results.md). The reasoning behind the wrong guess — that `icon-allow-overlap: true` disables culling so every symbol renders — had the mechanism right and the cost backwards: with collision detection off there is no per-frame placement work at all, just draw calls, which the GPU absorbs easily.

---

## Tier 3 — Features that open markets

Everything above restores what's already promised. These are genuinely new, and each is here because it unlocks something specific in [18-use-cases.md](18-use-cases.md) or [19-commercialization.md](19-commercialization.md) — not because it would be nice to have.

### `NW-015` — Multi-tenancy and organisation scoping · ~2–3 weeks

**Prerequisite for every SaaS pricing model in [19-commercialization.md](19-commercialization.md).** There is currently no tenant concept anywhere: one Postgres schema, one Kafka topic namespace per environment, one Keycloak realm, RBAC roles with no organisational boundary. Two customers cannot share a deployment.

That's survivable if you sell only self-hosted and edge appliances — and it may be the right call to *stay* single-tenant and lean into that. But it closes off hosted SaaS entirely, so the choice should be deliberate. The groundwork that exists: AOI-scoped WebSocket subscriptions (API-002) already prove per-connection data scoping works, and Postgres RLS is already in use on `audit_log`.

### `NW-016` — Usage metering · ~1 week

No service counts anything billable: not ingested events, not inference frames, not COA generations, not connected sensors. Any consumption-based or per-sensor price needs this, and retrofitting metering into a running deployment is far harder than building it now. `audit-svc` is the natural home.

### `NW-017` — Integrator self-service: API keys, rate limits, a docs portal · ~2 weeks

Epic 9 and persona 7 (Systems Integrator) describe a third-party developer authenticating with a scoped OAuth2 client-credentials key, subject to its own rate limits and revocation path (REQ-8.5). **None of that is built.** The public API is real and OpenAPI 3.1 specs are genuinely generated from the Zod schemas on six services — but the only way in is a human Keycloak JWT.

G8 commits to ≥3 third-party integrations live by GA. That goal is unreachable without this, and the API-as-a-product revenue line in [19-commercialization.md](19-commercialization.md) depends on it entirely.

### `NW-018` — Historical playback and pattern-of-life · ~2 weeks

`UC-2.2` ("query historical track data for a specific entity") is an Analyst's core workflow and has **no UI**. The data is there — every entity upsert flows to the durable `{env}.vektor.entities.upserted` Kafka topic added in Phase 6, and PostGIS holds the state. What's missing is a time-slider on the map and an entity-history view.

This is the single most-requested capability in every surveillance product in every vertical in [18-use-cases.md](18-use-cases.md), and it's a feature that demos extremely well.

**Correction (2026-10-16):** an earlier version of this entry called the Phase 6 durable-topic work "the hard half of it". That was optimistic and is wrong. `packages/db/src/schema/entities.ts` is current-state only — one row per entity, `last_updated` overwritten in place — and a Kafka topic can be replayed but cannot answer "where was entity X between 14:00 and 14:30" or "what did this AOI look like at 14:12", which is precisely what UC-2.2 and a time-slider need. The durable topic is the right *source*; something still has to project it into a time-indexed store. Scope this as: a new `entity_history` table, a consumer projecting `{env}.vektor.entities.upserted` into it, and a read API — plus a retention decision, because one entity at 10 updates/sec for 24h is ~864k rows.

### `NW-019` — Connector SDK plus three reference connectors · ~3 weeks

`ingest-svc` currently supports RTSP/SRT, GeoTIFF/SAR, AIS, ADS-B, MQTT, and the field-PWA HTTP endpoint, plus `mavlink-bridge` for MAVLink. Each was hand-written into the service. A documented adapter interface with three externally-contributed reference implementations would both satisfy G8 and make "does it support *my* sensor?" a one-day answer instead of a roadmap item. Obvious candidates: ONVIF cameras, LoRaWAN, and a generic Modbus/SCADA tap for the infrastructure vertical.

### `NW-020` — Simulation / wargaming mode · ~3 weeks

Already captured as a §7.1 post-v1 candidate. Promoted here because it has a commercial dimension the PRD didn't consider: **training is a revenue line, and you cannot train operators on live sensors.** Replaying a recorded or synthetic scenario against the real pipeline also solves demo-ability (`NW-004` needs something to show) and gives `NW-013` and `NW-014` their test harness. Three features, one build.

### `NW-021` — Custom model training as a product surface · ~2 weeks

`cv-train-svc` already does real YOLOv8 training and INT8 ONNX export, and `cv-inference-svc` already has a verified hot-swap upload endpoint (ML-006, tested with two genuinely different models). The pieces of "we'll train a detector on *your* objects and ship it to your deployment" exist and have never been joined into a workflow. [19-commercialization.md](19-commercialization.md) treats this as one of the highest-margin services available.

---

## Corrections to older docs found during this review

Three things in the existing docs are now out of date. Worth fixing in place so the next reader isn't misled:

1. **[09-roadmap.md](09-roadmap.md) still shows SEC-001's Istio `AuthorizationPolicy` as deliberately deferred.** It was implemented since — `vektor-infra` commit `5999548`, "Implement SEC-001's deferred Istio AuthorizationPolicy", and `AuthorizationPolicy` now appears in the istio module's `main.tf` and `variables.tf`.
2. **Project memory records "no service has a `deploy/helm` chart" as a systemic gap.** Eleven services now have one. The gap moved: the charts exist, the *images they reference* don't (`NW-001`/`NW-002`).
3. **[16-system-status.md](16-system-status.md) lists `field-pwa` as possibly still a compose placeholder.** It's a real Vite app with `src/lib` and `src/components`, and `vektor-web` commit `38262f2` added browser camera capture to it.

---

## A concrete first-30-days order

If one person is doing this, in this order:

| Days | Do | Why this order |
|---|---|---|
| 1 | `NW-011` (licence), `NW-005` (blue-force RBAC) | Both are hours, not days. One is a legal exposure, the other a security hole. Neither should survive the week. |
| 2–3 | `NW-003` (cloud compose) | Everything else gets faster once the stack comes up with one command. |
| 4–8 | `NW-001` (nine Dockerfiles) | The compose file from `NW-003` will have already clarified each service's runtime needs. |
| 9–10 | `NW-002` (publish images) | Makes `NW-001` consumable. |
| 11–15 | `NW-004` (one deployed instance) | The first time the system exists somewhere other than a laptop. |
| 16–18 | `NW-013` (measure the 60-second claim) | Now possible, because `NW-004` exists. Produces the number the whole pitch rests on. |
| 19–22 | `NW-006`+`NW-007` (CI coverage, fixture) | Protects everything above from regressing. |
| 23–30 | Order `NW-012`'s hardware, start `NW-018` (historical playback) while it ships | Hardware has lead time; start the long-pole purchase early and build the best demo feature while waiting. |

Tier 3 beyond that depends on which wedge you pick in [18-use-cases.md](18-use-cases.md) — the right next feature is genuinely different for maritime than for critical infrastructure, and committing to both at once is how a solo-built platform stalls.

---
