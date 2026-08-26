# ADR-0006: Real Debezium CDC, not a hand-rolled change-detection poller

**Status:** Accepted (Phase 5, 2026-08-25)

## Context

`logistics-svc` (SVC-017) needs to sync inventory state from an external ERP system into VEKTOR's own `inventory_items`/`inventory_history` within REQ-1.7's 60-second bound. The simplest thing to build is a polling loop that diffs the ERP table on an interval; the PRD instead specifies Debezium.

## Decision

Use a real Debezium `PostgresConnector` (pgoutput logical-replication plugin) registered via Kafka Connect against a dedicated "ERP" Postgres instance, producing genuine Kafka Connect JsonConverter envelopes that `src/cdc/inventoryConsumer.ts` decodes and applies.

## Consequences

- Change events are pushed via logical replication as they commit, not discovered by polling — inserts and updates land well within the 60s bound (verified end-to-end at "within seconds" in testing), and there's no missed-update window between poll intervals.
- The ERP Postgres instance must run with `wal_level=logical` — a non-default Postgres configuration that any real ERP integration target must also support and enable. This is an external-system constraint VEKTOR can document but not control.
- Kafka Connect is now a real infrastructure component in the deployment (not just Kafka + Postgres), with its own operational surface (connector registration, offset tracking, restart-on-failure behavior) that a hand-rolled poller wouldn't have had. Accepted as the right trade for guaranteed, replication-log-backed change capture rather than best-effort polling.
