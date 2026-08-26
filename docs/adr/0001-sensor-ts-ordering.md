# ADR-0001: `sensor_ts`, never `kafka_received_at`, as the fusion ordering key

**Status:** Accepted (Phase 1/3, 2026-08-21)

## Context

Sensors transmit out of order relative to each other due to variable network latency (satellite uplink, degraded RF links, buffered edge nodes). `fusion-svc`'s Kafka consumer has two candidate timestamps to order events by: `kafka_received_at` (when the broker saw the message) or `sensor_ts` (the GPS-derived, NTP-synced timestamp the sensor itself attached). See Pitfall 1, [10-risks-pitfalls.md](../10-risks-pitfalls.md).

## Decision

Every domain adapter (SVC-001/002/003) attaches `sensor_ts` at the point of capture, validated by `@vektor/kafka`'s `assertValidSensorTs` before the message is ever produced. `fusion-svc`'s `WatermarkWindow` (`@vektor/redis`) orders strictly by `sensor_ts` with a 500ms event-time watermark, and never falls back to arrival order.

## Consequences

- A moving entity never appears at two locations simultaneously due to reordering — the failure mode this ADR exists to prevent.
- The system has a hard dependency on sensor clock sync (NTP/PTP). A sensor with a badly skewed clock produces a badly ordered track, not an error — there's no runtime cross-check between `sensor_ts` and wall-clock time beyond schema validation. The NTP/PTP daemon itself (SVC-004) is host/infra config, not application code, and remains the operational mitigation for clock skew.
- Every future adapter (a domain not yet built) must attach a real `sensor_ts`, not a synthesized one, or it silently reintroduces the exact bug this ADR closes.
