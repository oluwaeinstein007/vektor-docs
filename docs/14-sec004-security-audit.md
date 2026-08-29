# VEKTOR — SEC-004: Penetration Testing & Dependency Audit

[← Back to VEKTOR-PRD.md](../VEKTOR-PRD.md)

---

## 22. Scope and What This Document Is

Roadmap SEC-004 calls for "Penetration testing + red team exercise." A genuine red-team exercise needs an authorized external firm, a live deployed target, and a signed rules-of-engagement document — none of which exist in this sandbox (no live cluster, no deployed environment; see [09-roadmap.md](09-roadmap.md) Phase 7 for the same constraint noted against PERF-002/003/004). What *is* deliverable directly, and is captured here:

1. **A real, current dependency vulnerability audit** (§23) — actual `pnpm audit` output against the live `vektor-backend` workspace lockfile, not a template.
2. **The engagement plan and rules of engagement** (§24) an external pentest firm or internal red team would execute against once a real environment exists.
3. **A self-audit against the SEC-003 auth boundary this same phase built** (§25) — the closest thing to an actual authenticated-attack test achievable without a live deployment: real forged/tampered/wrong-role tokens thrown at the real `requireRole()` implementation.

This is a point-in-time snapshot (2026-08-26). Dependency advisories change daily — re-run `pnpm audit` before treating §23's specific CVE list as current.

---

## 23. Dependency Vulnerability Audit (2026-08-26)

### 23.1 `vektor-backend` (`pnpm audit`, live workspace lockfile)

| Severity | Count |
|---|---|
| Critical | 1 |
| High | 18 |
| Moderate | 18 |
| Low | 1 |

**Critical — requires action:**

| Package | Path | Issue | Fix |
|---|---|---|---|
| `protobufjs` (<7.5.5) | `cv-inference-svc` → `onnxruntime-node` → `web` → `onnx-proto` → `protobufjs` | Arbitrary code execution (GHSA-xq3m-2v4x-88gg) | Upgrade transitive `protobufjs`, or confirm via `pnpm why protobufjs` whether onnxruntime-node's unused web-backend metadata dependency can be excluded from the install entirely (this project only uses the Node native execution provider, never the web backend) |

Note: this is **not** `vektor-proto`'s own gRPC code path — per [ADR-0005](adr/0005-cross-repo-grpc-wrapper.md), that code goes through `@grpc/grpc-js` and `@bufbuild/protobuf/wire`, not `protobufjs`. The critical finding is a transitive dependency of `onnxruntime-node` pulled in regardless of which execution provider is actually used at runtime — worth confirming whether it's reachable code at all before treating it as equally urgent as a directly-exercised path, but it still needs a tracked fix since "unreachable in this build" isn't the same guarantee as "patched."

**High — should be scheduled:**

| Package | Issue | Notes |
|---|---|---|
| ~~`drizzle-orm` (<0.45.2)~~ | ~~SQL injection via improperly escaped SQL identifiers (GHSA-gpj5-g38j-94v9)~~ | **Fixed.** Upgraded to `drizzle-orm@^0.45.2` (the actual npm `latest`, not a mid-range pin) + `drizzle-kit@^0.31.10` across all 8 direct consumers (`packages/db`, `geospatial-svc`, `fusion-svc`, `audit-svc`, `alert-svc`, `logistics-svc`, `reporting-svc`, `coa-svc`). Checked the real upstream changelogs for every version in the 0.39–0.45.2 range first — nothing touches `customType`/geometry, RLS, raw `sql`, or drizzle-kit's schema-config loading (the areas this project's own documented pitfalls depend on), so this wasn't a blind bump. Verified for real afterward: full workspace `pnpm build`/`pnpm lint` clean (18/18, 24/24), and `packages/db`'s RLS suite plus every drizzle-touching service's real-infra tests (`geospatial-svc`'s PostGIS bbox/ST_Contains, `fusion-svc`'s EKF/blue-force/no-strike-zone queries, `audit-svc`, `alert-svc`'s geofence/anomaly persistence) pass identically to pre-upgrade. `logistics-svc`/`coa-svc`/`reporting-svc` verified via build+lint only (their test suites need Debezium/pgRouting/Qdrant/headless-Chrome infra not exercised in this pass) — worth a follow-up full-suite run next time that infra is up. |
| `nodemailer` (multiple advisories) | SMTP command injection, header injection, SSRF via raw-option bypass, ReDoS in address parsing | Used in `alert-svc` (email dispatch) and `reporting-svc` (scheduled report delivery). No user-supplied input reaches nodemailer's `raw`/`envelope` options in this codebase — recipient addresses and subject lines are the only user-influenced fields, and neither is the vector for the raw-option/jsonTransport advisories. Still schedule the upgrade. |
| `@fastify/static` (multiple advisories) | Path traversal / route-guard bypass via encoded separators / non-canonical URLs | Used by `map-tile-server` (serving PMTiles) and transitively by every service's new `@fastify/swagger-ui` registration (API-001, this phase). `map-tile-server` is the higher-risk instance — it serves from a real filesystem directory; confirm its static-file root is scoped correctly and upgrade before any public-facing deployment. |
| `postcss`, `sharp`, `adm-zip`, `extract-zip`, `image-size` | Various DoS/path-traversal/file-read advisories | All build-tooling or image-processing transitive deps (`apps/web`'s Next.js toolchain, `reporting-svc`'s image handling). Lower priority — not on a path that processes untrusted external input directly in this codebase as currently built, but track for the next dependency-refresh pass. |

**Full machine-readable output**: re-run `pnpm audit --json` from `vektor-backend/` — not checked into this doc since it's already stale the moment a new advisory publishes.

### 23.2 Other repos

`vektor-proto` and `vektor-edge/scripts`'s audits could not be completed in this session — this sandbox's outbound connection to `registry.terraform.io` (and, intermittently, `registry.npmjs.org`) degraded to outright connection timeouts partway through this phase (the same environmental pattern already documented in [10-risks-pitfalls.md](10-risks-pitfalls.md)'s pitfall history for Phase 6's ARM64 builds — see `[[vektor-build-conventions]]`). Re-run `pnpm audit` in both once network access is confirmed healthy; do not assume they're clean just because they weren't flagged here.

`vektor-ml`'s two Python services were not scanned — `pip-audit` could not create its internal venv in this sandbox (`ensurepip` unavailable, no sudo to install `python3-venv`, the same constraint already documented for this project's own venv setup). A working pattern for Python dependency scanning here would need the same `--without-pip` + manual `get-pip.py` workaround already used for this project's own venvs, then a scanner that doesn't try to create a second nested venv — not attempted in this pass. **This is an open gap in this audit, not a clean bill of health for the Python side.**

---

## 24. Penetration Test Engagement Plan

For execution once a real staging or pre-production environment exists (not this local sandbox).

### 24.1 Scope

| In scope | Out of scope |
|---|---|
| Public REST API (`/api/v1/*` across all 6 REST-exposing services, §13.1) | Third-party infra (Kong/Keycloak/Kafka/Postgres internals themselves — vendor-patched, not this project's code) |
| WebSocket subscription gateway (`fusion-svc`, REQ-9.2) | Physical security of edge nodes (covered by §14.1's "Edge node physical compromise" mitigation, LUKS — SEC-005, separately tracked) |
| RBAC/JWT verification boundary (`@vektor/auth`, SEC-003) | Denial-of-service load testing (that's PERF-001's job, not a pentest's — the two shouldn't run concurrently against the same target) |
| Istio mTLS boundary once `AuthorizationPolicy` lands ([ADR-0012](adr/0012-istio-mtls-before-authz-policy.md)) | Social engineering, physical facility access |
| `vektor-ml` enclave network isolation (SEC-002) | The Keycloak/Kong control planes themselves (test *use* of them, not their own product security) |

### 24.2 Test categories (mapped to this project's actual controls)

1. **Authentication bypass** — forged/expired/wrong-algorithm JWTs against every `requireRole()`-gated endpoint; confirm §25's self-test results hold against the real deployed JWKS, not just the local-JWKS test fixture.
2. **Authorization boundary** — a token for one role attempting every other role's endpoints (e.g., an `Analyst` token against `POST /api/v1/coa/:coa_id/approve`); confirm the "+"-tier semantics in [`roles.ts`](../vektor-backend/packages/auth/src/roles.ts) match the intended access model, not just that *some* check exists.
3. **Injection** — prompt injection via sensor metadata/analyst notes into COA generation (Pitfall 4's `sanitizeForPrompt()`); SQL injection attempts against any raw `sql` fragment (audit for any future query that interpolates unescaped user input, given the `drizzle-orm` advisory in §23.1).
4. **Data classification boundary** — confirm an `Analyst` token genuinely cannot reach `SuperAdmin`-only data (`GET /api/v1/audit`) even via a parameter-tampering or IDOR-style attempt against a resource ID.
5. **Enclave isolation** — attempt lateral movement from a compromised `vektor-backend` pod into `vektor-ml-enclave` past the default-deny NetworkPolicy (§14.2, [ADR-0013](adr/0013-network-policy-default-deny.md)); attempt the same from a compromised enclave pod back out.
6. **Edge node** — attempt to extract credentials/data from a captured edge node once SEC-005 (LUKS full-disk encryption) exists; confirm the offline-sync intake queue doesn't leak unencrypted sensitive data to disk outside the encrypted volume.
7. **Rate limiting / API key scoping** — once Persona 7's (Systems Integrator) scoped API-key flow exists, confirm key revocation and rate limits actually hold under a real attempted-abuse pattern.

### 24.3 Rules of engagement (template — finalize with the actual firm/team before execution)

- Written authorization required before any test begins, scoped to a named non-production environment.
- No destructive testing against the shared dev Postgres/Kafka/Redis instances multiple engineers may be actively using (see the coordination pattern in `[[vektor-build-conventions]]` — the same "check with active sessions before touching shared infra" discipline applies to a red team as much as to a developer).
- Any finding rated High or Critical gets a same-day disclosure to the security lead, not held for a final report.
- Full findings feed back into [10-risks-pitfalls.md](10-risks-pitfalls.md)'s risk register and, for anything that represents a genuine architectural decision (not just a bug), a new [ADR](adr/README.md).

---

## 25. Self-Audit: SEC-003's Auth Boundary Under Adversarial Input

The closest thing to an actual attack test achievable without a live deployment: `packages/auth/tests/plugin.test.ts` and every service's route tests (this phase) throw real adversarial inputs at the real, unmocked `requireRole()` implementation — not a theoretical description of what *should* happen.

| Attack attempted | Result |
|---|---|
| No `Authorization` header | 401, real assertion, passes |
| Tampered signature (valid JWT with 3 characters altered) | 401 — `jose`'s real RS256 verification rejects it, not a mocked check |
| Token signed by a different keypair entirely (not in the service's JWKS) | 401 |
| Valid token, role below the endpoint's requirement (e.g. `Viewer` against `analyst+`) | 403 |
| Valid token, role satisfies requirement via the documented `SuperAdmin`/`Commander` override | 200 — confirms the override is intentional, not a bypass |
| Valid token missing any recognized role in `realm_access.roles` | 401, not a crash or a default-allow |
| Client-supplied `actor_user_id`/`actor_role`/`operator_id` body fields on an authenticated write | Rejected/ignored by schema — identity is read from the verified token only ([ADR-0009](adr/0009-actor-identity-from-verified-jwt.md)), closing a real spoofing vector that existed before this phase |

All of the above are real, passing, checked-in tests (152 tests total across the 7 packages this phase touched, all green as of 2026-08-26) — not claims made without execution.

**What this self-audit does *not* cover**: the live Keycloak realm's own configuration (client secrets, token lifetimes, refresh-token rotation policy), rate limiting at the Kong gateway layer, or anything requiring a real network-reachable deployment. Those remain §24's job once a target environment exists.
