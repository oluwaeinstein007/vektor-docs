# ADR-0011: Postgres RLS as a DB-layer backstop for the audit log, on top of app-layer RBAC

**Status:** Accepted (Phase 7 / SEC-003, 2026-08-26)

## Context

`audit_log`'s append-only guarantee (REQ-8.3) was originally enforced purely by convention: `audit-svc` simply never implements an UPDATE or DELETE query against that table. §14.1's threat model calls for "RBAC + row-level security" together, not RBAC alone, for unauthorized-data-access mitigation.

## Decision

Enable Postgres row-level security on `audit_log` with policies scoped to a `vektor_app` role: unrestricted SELECT/INSERT, and UPDATE/DELETE policies that evaluate `using: false` unconditionally — combined with *no* UPDATE/DELETE grant to that role at all, so the guarantee holds even if a future bug granted those privileges by mistake.

## Consequences

- Verified directly against a real, non-superuser `vektor_app` Postgres connection: SELECT/INSERT succeed, UPDATE/DELETE fail with `permission denied` — this is a real, working DB-level control, not a syntactically-valid-but-unverified migration.
- **Load-bearing caveat, not yet closed**: RLS never applies to a table's owner or to a superuser connection, and every service's `DATABASE_URL` in this repo's current dev/test configuration connects as the `postgres` superuser. This policy is currently inert in practice until each service's connection identity is switched to `vektor_app` (or an equivalent non-owner, non-superuser role) as a separate infra/secrets rollout — this ADR authorizes the DB-side control, it doesn't complete the deployment change that activates it.
- `vektor_app` itself is modeled as `pgRole("vektor_app").existing()` in the Drizzle schema — a cluster-level object provisioned once via infra/Vault-managed credentials, not created by an application migration that would attempt to recreate it on every fresh environment.
