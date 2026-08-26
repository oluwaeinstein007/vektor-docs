# ADR-0009: Actor identity for audit/approval actions comes from the verified JWT, never a request body field

**Status:** Accepted (Phase 7 / SEC-003, 2026-08-26)

## Context

Before SEC-003 wired up RBAC, every route that needed to know "who did this" (COA approve/reject, alert acknowledge) took `actor_user_id`/`actor_role` (or `operator_id`) as plain fields in the request body — the only option available before any auth middleware existed to populate a verified identity. This was explicitly flagged as temporary in the original route comments ("routes trust the caller pending SEC-003").

## Decision

Once `@vektor/auth`'s `requireRole()` preHandler populates `request.user` from a verified JWT, every route that writes an audit entry or records "who acted" reads `request.user.sub`/`request.user.role` — the client-supplied `actor_user_id`/`actor_role`/`operator_id` body fields are removed from the corresponding Zod schemas entirely, not just ignored.

## Consequences

- A caller can no longer approve a COA, acknowledge an alert, or trigger any RBAC-gated action while having the audit log attribute it to an arbitrary asserted identity — closing a real spoofing gap, not just a theoretical one (the fields existed and were wired into `audit-svc` writes verbatim).
- Removing the fields from the Zod schema (rather than leaving them present-but-ignored) means a client that still sends them gets no error and no effect — Zod's default `.object()` behavior strips unknown-but-declared-elsewhere keys silently. This is intentional: a deprecation/removal, not a soft transition.
- `audit-svc`'s own `POST /api/v1/audit` write endpoint remains ungated by `requireRole()` — its caller is another *service*, not a human with one of §5.1's roles, so the actor-identity fix doesn't apply to that hop. The real boundary for "is this caller allowed to write audit entries" there is service-identity (mTLS, ADR-0012), not user role; this is a separate, still-open concern from the one this ADR closes.
