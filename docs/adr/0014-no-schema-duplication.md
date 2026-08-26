# ADR-0014: Every wire contract is defined exactly once, in `vektor-proto`

**Status:** Accepted (Phase 0)

## Context

VEKTOR is a 6-repo polyrepo (§9). Without a single source of truth for shared data shapes, a Kafka event schema, a REST response shape, and a Socket.io event payload could each drift into their own hand-maintained copy across `vektor-platform`, `vektor-ml`, and `apps/web` — exactly R-006/R-010's risk (schema drift breaking consumers; polyrepo overhead slowing cross-cutting changes).

## Decision

Every Zod contract shape is defined exactly once, in `vektor-proto/src/schemas/*.ts`. Every consuming package (`packages/shared`, `packages/kafka`, `packages/qdrant`, etc.) only ever re-exports (`export * from "@vektor/proto"` or a named re-export) — never redeclares. The same discipline extends to the one TS↔Python crossing: `llm.proto` is the single source for the gRPC contract, never hand-copied into either consuming repo (Pitfall 3).

## Consequences

- A breaking contract change is a single PR in `vektor-proto` with dual-team review (§9.4/§9.9), then a version bump in each consumer — mechanical, not a manual hunt across repos for every place a shape got copied.
- This discipline has needed real enforcement, not just initial design intent: `docs/07-data-api.md` itself was found stale and showing hand-rolled redefinitions (fixed 2026-08-21), and a storage-layer Drizzle row was found *not* going through `Entity.parse()` before reaching a response, silently violating REQ-9.1's "100% of public endpoints ... schema-validated" (fixed by adding a proper row→`Entity` mapper). The rule is easy to state and easy to accidentally violate at the boundary where a raw DB row or a hand-built object gets returned without being parsed through the canonical schema — every new route/mapper should be checked against this specifically, not assumed compliant by default.
- `packages/shared`/`packages/kafka`/`packages/qdrant` importing a library that a sibling `@vektor/*` package already depends on must still declare that library as its own direct dependency — pnpm's strict `node_modules` isolation doesn't hoist a dependency's own dependencies into a package that merely depends on that package.
