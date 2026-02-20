# PostgreSQL Replication Research

## Overview
PostgreSQL replication involves copying data from one server (primary) to one or more other servers (standbys). This provides high availability, disaster recovery, and read scalability.

## Replication Types
- **Physical Replication (Streaming Replication):** Replicates entire databases byte-by-byte via streaming WAL segments.
- **Logical Replication:** Replicates individual tables, schemas, or databases based on changed data. Allows cross-version replication and selective data sharing.
- **Synchronous Replication:** The primary waits for the standby to confirm receipt and write of WAL before committing. Guarantees no data loss but can increase latency.
- **Asynchronous Replication (Default):** The primary commits immediately without waiting for the standby. Low performance impact, but small risk of data loss on primary failure.

## High Availability (HA) Solutions
- **Patroni:** Modern, automated HA management for PostgreSQL using a distributed configuration store (e.g., etcd, Consul, ZooKeeper).
- **repmgr:** A management tool for streaming replication and automated failover.
- **HAProxy / PgBouncer:** Load balancers that route connections to the current primary and/or secondaries.

## Replication Slots
- **Physical Replication Slots:** Ensures the primary doesn't delete WAL segments until they are consumed by the standby.
- **Logical Replication Slots:** Tracks progress for logical replication consumers.

## Monitoring Replication
- **`pg_stat_replication`:** View replication status on the primary.
- **`pg_stat_wal_receiver`:** View replication status on the standby.
- **Replication Lag:** Measure the distance between the primary and standby in bytes or time.

## Disaster Recovery Strategies
- **Multi-Site Replication:** Replicating to a standby in a geographically separate data center or cloud region.
- **Continuous Archiving:** Using WAL archives to allow point-in-time recovery on the primary or a new server.

## Best Practices (2024-2025)
- **Use Replication Slots:** To prevent standbys from falling behind and losing WAL data.
- **Hot Standby:** Allow read-only queries on the standby server to offload the primary.
- **Monitoring Tools:** Use Grafana or PMM for real-time replication lag dashboards.
- **Automated Failover:** Implement Patroni for reliable, production-ready HA.
- **WAL Archiving:** Always enable WAL archiving as a second line of defense for DR and PITR.

## Key SQL Queries
- `SELECT * FROM pg_stat_replication;`
- `SELECT pg_current_wal_lsn(), pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn();`
- `SELECT slot_name, slot_type, active, restart_lsn FROM pg_get_replication_slots();`

## References
- [PostgreSQL Documentation: High Availability and Replication](https://www.postgresql.org/docs/current/high-availability.html)
- [Patroni: Documentation](https://patroni.readthedocs.io/en/latest/)
- [pganalyze: Guide to PostgreSQL Replication](https://pganalyze.com/blog/postgresql-replication-slots)
