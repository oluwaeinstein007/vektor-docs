# ADR-0005: Runtime values crossing the `vektor-proto` `link:` boundary go through a wrapper module

**Status:** Accepted (Phase 4, 2026-08-24)

## Context

`vektor-proto` is consumed from `vektor-platform` via `link:../../../vektor-proto` — a filesystem symlink, not a published package, in local dev. This means `vektor-proto` carries its own, entirely separate `node_modules` tree. `vektor-proto/src/generated/llm.ts` (ts-proto output) is built against *its own* `@grpc/grpc-js` install. When `coa-svc` (ML-010) imported `@grpc/grpc-js` directly and constructed `credentials.createInsecure()` to pass into a class from the generated code, grpc-js's internal `instanceof ChannelCredentials` check failed — even at the identical package version in both installs, they're two distinct module instances, and `instanceof` is instance-identity-sensitive. The failure mode wasn't consistent: a synchronous `TypeError` in one call context, a silent 0%-CPU hang in another (inside `node:test`'s runner).

## Decision

`vektor-proto` exports a hand-written wrapper (`vektor-proto/src/llm.ts`) re-exporting both its generated types *and* the specific `@grpc/grpc-js` runtime values (`credentials`, `Server`, `ServerCredentials`) a consumer needs to construct objects that will be passed into that generated code. Every consumer — `coa-svc`, `@vektor/shared/llm`'s re-export, every test file's fake gRPC server — imports these runtime values from the wrapper, never directly from its own `@grpc/grpc-js`.

## Consequences

- Type-only imports (`sendUnaryData`, `ServerUnaryCall`, `CallOptions`) are unaffected by this — TS types have no runtime instance identity, so a consumer's own `@grpc/grpc-js` import is fine for types.
- This is the general rule for any future `link:`-boundary package that a consumer needs to *construct* runtime objects for (not just pass plain data across): identify every runtime class the generated code's API surface takes as input, and add each to the wrapper rather than trusting version-pinning alone to make cross-install instances interchangeable.
- Once `vektor-proto` moves from `link:` to a real published/versioned package (its stated production path), this specific class-identity problem may or may not still apply depending on the package manager's dedup behavior — re-verify rather than assuming the wrapper becomes unnecessary.
