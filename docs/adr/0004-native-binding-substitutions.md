# ADR-0004: Substitute pure-JS/algorithmic alternatives when a native binding has no prebuilt binary

**Status:** Accepted (Phase 1/2, 2026-08-21–23)

## Context

Two PRD-specified libraries turned out to require a full native source build with no prebuilt binary available for this project's Node ABI/sandbox: `gdal-async` (SVC-002, GeoTIFF/SAR ingestion) and `opencv4nodejs` (ML-005, optical-flow velocity estimation). Both need network access to fetch build toolchains/headers that this environment doesn't reliably have.

## Decision

- SVC-002 uses `geotiff` (pure JS) instead of `gdal-async` for the GeoTIFF/SAR pipeline.
- ML-005 estimates velocity/heading via the same Kalman-filter state (ADR-0002/0003) already being tracked, instead of `opencv4nodejs`'s optical flow. REQ-2.5 explicitly allows either path, so this isn't a scope reduction.

## Consequences

- Both services build and run in environments without a C++ toolchain or network access to `nodejs.org`'s header archive — a real constraint in this sandbox, and plausibly in a locked-down edge/air-gapped deployment too (§11.1).
- `geotiff` may have narrower format/codec coverage than GDAL's full format library — revisit if a future sensor feed uses a GeoTIFF variant `geotiff` doesn't parse.
- The Kalman-filter velocity path only ever produces a *derived* estimate from already-tracked positions; it can't independently corroborate motion the way true optical flow could (e.g. distinguishing a genuinely stationary object with camera shake from a truly moving one at very low pixel displacement). Acceptable per REQ-2.5, but worth knowing if a future accuracy regression traces back to velocity estimation specifically.
- General principle for future services: when a native binding can't be installed in the target build environment, prefer a pure-JS or already-available-primitive substitution over blocking the roadmap task — document the substitution and the concrete reason (no prebuilt binary + no network to source-build), not just "couldn't install X."
