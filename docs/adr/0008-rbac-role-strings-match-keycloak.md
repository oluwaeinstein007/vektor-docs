# ADR-0008: RBAC role strings match Keycloak's literal role names, spaces included

**Status:** Accepted (Phase 7 / SEC-003, 2026-08-26)

## Context

§5.1's RBAC Role column names six roles: `Commander`, `Analyst`, `Logistics Officer`, `Field Operator`, `SuperAdmin`, `Viewer` — two of them (`Logistics Officer`, `Field Operator`) contain a literal space. While building `@vektor/auth`'s role enum, the natural instinct was to normalize these to PascalCase identifiers (`LogisticsOfficer`, `FieldOperator`) for a cleaner TypeScript union type. A live Keycloak "vektor" realm (already provisioned in this environment for demo purposes) was checked before shipping, and its actual role names are `Logistics Officer` and `Field Operator` — spaces included, matching the PRD's literal spelling, not the normalized one.

## Decision

`@vektor/auth`'s `Role` type and `ROLES` array use the exact §5.1 strings, spaces included. `requireRole()`'s role-requirement sets (`"logistics+"`, etc.) reference the same literal strings. No normalization/mapping layer sits between a Keycloak `realm_access.roles` claim and this type.

## Consequences

- A JWT verification path against the real Keycloak realm needs no translation step — the string comparison is a direct match.
- Every future reference to these two roles anywhere in the codebase (route guards, tests, seed scripts) must use the space-containing literal, not a camelCase/PascalCase guess — this is an easy, silent mismatch to introduce (`"LogisticsOfficer" !== "Logistics Officer"` fails with no type error, since both are valid-looking string literals until checked against the actual `Role` union).
- This was caught only because a live Keycloak instance happened to be available to check against during this session — a future role addition should verify against the actual identity provider's configuration before assuming a "cleaner" spelling, not just re-read the PRD table (which is easy to eye-skim past the space).
