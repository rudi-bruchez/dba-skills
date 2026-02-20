---
name: postgresql-replication
description: Expert skill for managing PostgreSQL replication, high availability, and disaster recovery.
version: 1.0.0
tags:
  - postgresql
  - dba
  - replication
  - high-availability
  - disaster-recovery
---

# PostgreSQL Replication

This skill provides expert guidance for choosing the right replication type, setting up standbys, and monitoring for replication lag in PostgreSQL.

## Core Capabilities

- **Replication Type Selection:** Choosing between physical (streaming) and logical replication.
- **High Availability (HA) Solutions:** Implementing Patroni, repmgr, and HAProxy for automated failover.
- **Replication Slots:** Using physical and logical replication slots to ensure data durability.
- **Monitoring Replication:** Tracking replication lag and status on the primary and standby.
- **Disaster Recovery (DR):** Implementing strategies for offsite storage and business continuity.

## Workflow: Replication Strategy

1.  **Define RPO & RTO:** Determine the maximum tolerable data loss (RPO) and downtime (RTO).
2.  **Select Replication Method:**
    - **Physical (Streaming):** For full cluster replication, high availability, and read scalability.
    - **Logical:** For individual database or schema replication, cross-version replication, and selective data sharing.
3.  **Use Replication Slots:** To prevent standbys from falling behind and losing WAL data.
4.  **Monitor Replication Lag:** In bytes or time, and set up alerts for high lag.
5.  **Automated Failover:** Implement Patroni for reliable, production-ready HA.
6.  **Offsite Storage:** Store WAL archives and physical backups in a separate location.

## Essential Commands

### 1. Check Replication Status (Primary)
```sql
SELECT * FROM pg_stat_replication;
```

### 2. Check Replication Status (Standby)
```sql
SELECT * FROM pg_stat_wal_receiver;
```

### 3. Measure Replication Lag (Primary)
```sql
SELECT
    application_name,
    client_addr,
    state,
    (pg_current_wal_lsn() - replay_lsn) AS lag_bytes,
    (pg_current_wal_lsn() - replay_lsn) / 1024 / 1024 AS lag_mb
FROM pg_stat_replication;
```

### 4. Use Replication Slots
```sql
-- Create a physical replication slot on the primary
SELECT pg_create_physical_replication_slot('standby_1');

-- View replication slots
SELECT * FROM pg_get_replication_slots();
```

## Best Practices (2024-2025)

- **Use Replication Slots:** To ensure the primary doesn't delete WAL segments until they are consumed by the standby.
- **Hot Standby:** Allow read-only queries on the standby server to offload the primary.
- **Synchronous Replication:** For zero-data-loss configurations, but be aware of the performance impact.
- **Monitoring Tools:** Use Grafana or PMM for real-time replication lag dashboards.
- **Automated Failover:** Implement Patroni for reliable, production-ready HA.
- **WAL Archiving:** Always enable WAL archiving as a second line of defense for DR and PITR.

## References

- [Failover Runbook](./references/failover-runbook.md)
- [Replication Diagnostics](./references/replication-diagnostics.md)
- [HA Architectures](./references/ha-architectures.md)
---
