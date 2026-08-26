# ADR-0007: pgRouting on a dedicated Postgres instance, not the main PostGIS database

**Status:** Accepted (Phase 5, 2026-08-25)

## Context

`logistics-svc`'s route optimization (SVC-019) needs `pgr_dijkstra`, which requires the `pgrouting` extension. The main PostGIS image this project standardizes on (`postgis/postgis:16-3.4-alpine`, used for `entities`/`coas`/`audit_log`/etc.) doesn't have `pgrouting` installed.

## Decision

Run a separate, dedicated pgRouting-extension Postgres instance for the road-graph tables (`road_nodes`, `road_edges`) and route queries, rather than switching the main database image or bolting the extension onto the shared instance.

## Consequences

- The main PostGIS instance's image and extension set stay unchanged for every other service — no risk of a `pgrouting` install issue or version conflict affecting `entities`/`coas`/anything else.
- `logistics-svc` now holds two separate database connections (`DATABASE_URL` for the shared schema, `ROUTING_DATABASE_URL` for the routing graph) rather than one — every new environment (dev, CI, prod) needs to provision and wire both.
- `pgr_dijkstra` (and likely other overloaded pgRouting/PostGIS functions) can't resolve its overload when every argument arrives as an untyped `postgres.js` parameter — calls need explicit casts (`${edgesSql}::text, ${startId}::bigint, ...`) or they fail with "function ... is not unique." Apply this whenever adding a new pgRouting call against this instance.
- A flagged/blocked road edge is modeled as an effectively-infinite-cost edge in the same graph, not a separate "blocked edges" table — so REQ-6.2's "recomputed within 5s of a new hazard flag" is automatic (every route request already reads the live `blocked` flag; there's no separate recompute step to schedule or fall behind on).
