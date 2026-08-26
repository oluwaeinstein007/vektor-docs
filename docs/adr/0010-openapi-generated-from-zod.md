# ADR-0010: OpenAPI 3.1 is generated from existing Fastify/Zod schemas, not hand-authored

**Status:** Accepted (Phase 7 / API-001, 2026-08-26)

## Context

REQ-9.1 requires a versioned public REST API with an OpenAPI 3.1 spec, "100% of public endpoints documented and schema-validated." Every route in every service already declares its request/response shapes as Zod schemas, enforced at runtime by `fastify-type-provider-zod`'s validator/serializer compilers (this is the existing enforcement mechanism for "100% ... schema-validated," predating this ADR). A hand-written OpenAPI YAML/JSON file was the alternative.

## Decision

Each public-facing service registers `@fastify/swagger` with `fastify-type-provider-zod`'s `jsonSchemaTransform`, generating the OpenAPI 3.1 document directly from the same Zod schemas the route handlers already validate against, exposed via Swagger UI at `/documentation`.

## Consequences

- The spec cannot drift from the actual validation behavior — there is only one schema per route, not a runtime one and a separately-maintained documentation one that can silently diverge.
- A route change that updates its Zod schema automatically updates its published spec on the next deploy; no separate "update the OpenAPI file" step to forget.
- `map-tile-server` was not given this treatment — it has no Zod-typed routes (a single `/healthz` and `@fastify/static` registration for tile serving), so there's no schema for the transform to generate from. If it later grows a real API surface, it should adopt the same pattern rather than a hand-written spec, for consistency.
- This covers REST only. REQ-9.1's sibling requirement, REQ-9.2 (WebSocket subscription API scoped to a bounding box, `fusion-svc`'s Socket.io gateway), has no OpenAPI equivalent — its contract lives in `vektor-proto/src/schemas/socket-events.ts` per §13.2, documented separately.
