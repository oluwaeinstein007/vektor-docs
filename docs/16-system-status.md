# VEKTOR — Current System Status

[← Back to VEKTOR-PRD.md](../VEKTOR-PRD.md)

---

**This is a living, dated snapshot, not part of the original PRD numbering.** This project has been under continuous, often highly concurrent (dozens of parallel Claude Code sessions on the same checkout at peak) development well past its original Phase 0–7 roadmap. Reconstructing "what's actually true right now" from `09-roadmap.md` plus scattered session memory has repeatedly cost real time across sessions — this doc exists to make that a 2-minute read instead. **Update it whenever you land a nontrivial change**, and trust the date on whatever you're reading: a status doc is a snapshot, not a promise.

*Last updated: 2026-10-16.*

**Looking for what to do next, rather than what exists?** [17-next-work.md](17-next-work.md) is the post-roadmap backlog, [18-use-cases.md](18-use-cases.md) ranks the candidate markets, and [19-commercialization.md](19-commercialization.md) covers licensing, pricing and the path to first revenue (all three written 2026-10-16).

## Repo layout (current)

7 repos, siblings under one parent folder (not itself a git repo):

| Repo | Contents |
|---|---|
| `vektor-proto` | Source-of-truth Zod schemas + `llm.proto` (the only TS↔Python crossing) |
| `vektor-backend` | All TS services + backend packages (was `vektor-platform` until the 2026-08-26 split — see `04-system-design.md` and the split's own commit for why) |
| `vektor-web` | `apps/web` (operator dashboard) + `apps/field-pwa` (browser-based GPS/gyro/camera field client) + `packages/ui` |
| `vektor-ml` | `cv-train-svc` (Python, YOLOv8 training), `llm-cloud-svc` (Python, vLLM) — the only Python in the system |
| `vektor-infra` | Terraform (K8s/Helm/Vault/Istio) |
| `vektor-edge` | Edge bootstrap scripts, offline PMTiles tooling, the edge Docker Compose overlay |
| `vektor-docs` | This repo |

## What's built and verified

**The entire original roadmap (Phases 0–7, `09-roadmap.md`) is complete** — ingestion (5 adapter types), CV/tracking, data fusion (EKF + cross-sensor correlation), decision support (RAG + LLM COA generation), logistics/alerting/reporting, edge deployment tooling, and public API/RBAC/security hardening. Every phase was verified against real infrastructure (real Postgres/PostGIS, real Kafka, real Redis, real Qdrant, a real headless browser) rather than mocks — the concrete pitfalls each phase surfaced (auth/CJS-interop/timing bugs and the like) live in each service's own README/code comments and in Claude session memory for this project, not in a single checked-in doc. The only gaps left from the original plan are hardware-only (real GPU/Jetson throughput — no such hardware exists in any sandbox this has been built in) and SEC-005 (edge-node disk encryption, not yet attempted).

**Beyond the original roadmap**, real device-integration and UI work has continued:

- **Phone-camera RTSP** — `ingest-svc`'s `RTSP_ROTATION` fix for orientation-less phone RTSP output (see `vektor_phone_rtsp_gotchas.md` project memory); verified end-to-end against a real Android phone running IP Webcam.
- **MAVLink bridge** (`services/mavlink-bridge`) — fuses drone/UGV MAVLink telemetry (real or SITL) into `fusion-svc` as a friendly track, via the same `IotTelemetryEvent` contract as any other IoT device (discriminated by `payload.device_class`).
- **Real public AIS feed** — a live Kystverket (Norwegian coastal) AIS feed ingested through the real pipeline, not synthetic data.
- **`apps/field-pwa`** — a browser-based field client (GPS/gyro, optional browser-camera capture as an RTSP fallback) posting to `ingest-svc`'s field-ingest HTTP endpoint.
- **Live camera feed panel** on the dashboard, fed by either a real RTSP source or field-pwa's browser-camera snapshots.
- **Target Workbench UI + `@vektor/ui` design system** in `apps/web` (Phase 4's COA/threat-scoring backend never had a matching UI until this).
- **Sensor registry + Sensor Coverage map layer** (2026-09-15) — `fusion-svc`'s `POST/GET /api/v1/sensors`/`DELETE /api/v1/sensors/:id` (REQ-1.8, Epic 1 — `docs/07-data-api.md` §13.1), the one requirements-table endpoint left unbuilt after Phase 1 closed. Registering a sensor now makes its `sensor:status` events carry real position/coverage-radius geometry, which the dashboard's "Sensor Coverage" layer renders as a client-side geodesic circle — previously a permanent no-op since Phase 1.
- **In progress as of this writing**: real role-gated navigation + an Admin Console and Logistics Workspace screen in `apps/web`, being built by a concurrent session — check `git log` in `vektor-web` for current status before assuming it's done or not started.

## Bringing up a local dev stack

**Updated 2026-10-16 (NW-003): there is now a cloud-profile compose stack.** From `vektor-backend/`:

```sh
pnpm stack:up          # core: postgis, kafka, redis, keycloak, maildev, mosquitto, mediamtx
pnpm stack:up:full     # + qdrant, routing-postgres, erp-postgres, debezium
cp compose/.env.example .env && set -a && . ./.env && set +a
```

`stack:up` starts the containers, waits for every healthcheck, and applies the Drizzle migrations. Host ports are deliberately unusual (postgres 15432, kafka 19092, redis 16379, keycloak 18080) because every port the service READMEs document was already taken by unrelated projects on a shared dev box — see `vektor-backend/compose/README.md`, which also records the four real bring-up bugs this surfaced and fixed. The old advice (bring each container up by hand from its service README) still works but is no longer necessary.

The compose stack runs a **real Keycloak** with the six §5.1 roles and one user per role imported (`pnpm stack:token Commander` prints a real access token), so the dev-auth-bridge route below is now the lighter alternative rather than the only option.

**As of 2026-09-15, `fusion-svc` refuses to start without `KEYCLOAK_JWKS_URI` set** — `packages/auth`'s `authOptionsFromEnv()` throws at startup if it's unset (`GET /api/v1/sensors` etc. are now real-RBAC-gated, not just flagged as a gap). For local dev, don't stand up real Keycloak: run the dev-only auth bridge instead (`pnpm --filter @vektor/auth dev:auth-bridge` from `vektor-backend/`, serves `http://localhost:4477/certs`) and point `KEYCLOAK_JWKS_URI=http://localhost:4477/certs` at it. See `vektor-web/apps/web/README.md`'s auth-bridge section for the matching frontend-side setup (mint a token via `/settings` → "Mint via dev bridge"). Never point a real `KEYCLOAK_JWKS_URI` at this bridge outside local development. ~~`blueforce/routes.ts` (blue-force-assets, no-strike-zones) does not have this gate yet~~ — **fixed 2026-10-16 (NW-005)**: all seven blue-force/no-strike routes are now gated (reads `all`, writes `commander`), `vektor-web`'s no-strike layer switched from a bare `fetch` to the authenticated client, and four new tests cover it. Every route surface in the repo is now behind `requireRole()`.

**Real gotcha, confirmed 2026-09-14**: dev infra containers persist across sessions (`docker start`, not `docker run`, once they already exist) — but a persisted Postgres data volume keeps whatever password it was *first* initialized with; `POSTGRES_PASSWORD` on a later `docker run`/`docker start` is silently ignored. If a service gets `password authentication failed` over its TCP `DATABASE_URL` connection while a `docker exec <container> psql` sanity check works fine, that's why — `psql` via `docker exec` hits pg_hba.conf's `local`/`127.0.0.1`/`::1` `trust` rule (no password check at all, a false-negative test), while any TCP connection from the host's Docker bridge IP hits a real `scram-sha-256` check. Fix once, at the container: `docker exec <container> psql -U postgres -d vektor -c "ALTER USER postgres WITH PASSWORD '<whatever every service's DATABASE_URL already assumes>';"`.

Also expect **container name/port collisions with completely unrelated projects** on a shared dev machine — `vektor-dev-postgis`'s usual host port can be squatted by another project's container. Check `docker ps -a` for what's actually free before assuming a documented port is available.

See `demo-screenshots/README.md` (repo root, sibling to all 7 repos) for a running gallery of real, captured proof that the live dashboard/pipeline works, plus the exact commands used to reproduce each shot.

## Multi-session coordination

This checkout regularly has multiple (sometimes dozens of) concurrent Claude Code sessions active. Before running `pnpm install` at a workspace root, restarting shared dev infra, or starting a service that owns a Kafka consumer group other sessions might also be running, use `ListAgents` and `SendMessage` to check first — a file showing "modified" in `git status` may be another session's in-progress work, not stale local state. This has caused real near-collisions (two sessions independently starting the identical dev stack, duplicate task assignments from the user across sessions) that were only avoided by checking first. See project memory (`vektor_project_overview.md`, `vektor_build_conventions.md`) for the specific incidents.

## Known open gaps (as of this writing)

- SEC-005 (edge full-disk encryption) — not attempted.
- PERF-002/003/004 and real GPU/Jetson throughput generally — hardware-only, no such hardware exists in any sandbox this has been built in.
- `field-pwa` didn't exist as an app until the device-integration work above — check its current state before assuming EDGE-001's `docker-compose.edge.yml` placeholder entry has been replaced with a real build.
- `coa-svc`'s edge mode still needs a full Postgres+Qdrant despite §11.4's edge stack table only listing `better-sqlite3` — documented architecture gap, not fixed.
- ~~The 469MB GGUF test fixture in `coa-svc` exceeds GitHub's 100MB limit~~ — **not actually open** (verified 2026-10-16): it is committed as a 134-byte Git LFS pointer, `.gitattributes` carries `*.gguf filter=lfs`, and `git lfs fsck` is clean against the pushed remote.
- **New as of 2026-10-16**: every service now has a Dockerfile (was 3 of 12), a `release-images.yml` workflow publishes them to GHCR, all seven repos carry a licence (open-core: Apache-2.0 / BSL 1.1), and `pnpm test:coverage` measures coverage for the first time. See [17-next-work.md](17-next-work.md)'s status section for what landed and the seven latent bugs it surfaced.
