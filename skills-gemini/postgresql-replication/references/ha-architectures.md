# HA Architectures

A guide to the most common High Availability (HA) architectures for PostgreSQL.

## 1. Single Primary with Multiple Standbys (Default)
A single primary server handles all writes, and one or more standby servers handle all reads and act as a backup.
- **Pros:** Simple to set up and manage.
- **Cons:** Manual failover and potential data loss if not using synchronous replication.

## 2. Streaming Replication with Patroni (Automated)
Patroni automatically manages a cluster of PostgreSQL servers, including automated failover and re-configuration.
- **Pros:** Reliable, production-ready, and highly customizable.
- **Cons:** Requires a distributed configuration store (e.g., etcd, Consul, ZooKeeper).

## 3. Streaming Replication with repmgr (Automated)
A management tool for streaming replication and automated failover.
- **Pros:** Simpler to set up than Patroni.
- **Cons:** Not as feature-rich as Patroni.

## 4. Logical Replication for Cross-Version/Selective Data Sharing
Logical replication allows for individual database or schema replication, cross-version replication, and selective data sharing.
- **Pros:** Flexible for many use cases.
- **Cons:** Not as fast as streaming replication for full cluster replication.

## 5. Multi-Site Replication for Disaster Recovery
Replicate data to a standby server in a geographically separate data center or cloud region.
- **Pros:** Provides business continuity in the event of a site-wide failure.
- **Cons:** Requires careful network and storage configuration.

## 6. Load Balancing with HAProxy and PgBouncer
Use HAProxy or PgBouncer to route client connections to the current primary and/or secondaries.
- **Pros:** Improves application performance and availability.
- **Cons:** Adds complexity to the overall architecture.

## References
- [PostgreSQL Documentation: High Availability and Replication](https://www.postgresql.org/docs/current/high-availability.html)
- [Patroni: Documentation](https://patroni.readthedocs.io/en/latest/)
- [repmgr: Documentation](https://repmgr.org/docs/current/)
- [pganalyze: Guide to PostgreSQL HA Architectures](https://pganalyze.com/blog/postgresql-ha-architectures)
---
