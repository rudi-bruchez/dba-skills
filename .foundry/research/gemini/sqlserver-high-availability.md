# SQL Server High Availability (HA) & Disaster Recovery (DR) Research

## Overview
SQL Server provides several options for High Availability (HA) and Disaster Recovery (DR). HA focuses on maximum uptime and automatic failover within a single site; DR focuses on recovery at a secondary site after a major failure.

## High Availability Options
- **Always On Availability Groups (AGs):** Modern, database-level HA. Replicates data across multiple nodes. Can be synchronous (no data loss) or asynchronous (low performance impact).
- **Failover Cluster Instances (FCIs):** Instance-level HA. Requires shared storage (SAN or S2D). Replaces an entire instance if a node fails.
- **Log Shipping:** Database-level DR. Asynchronous replication of transaction logs to a standby server. Manual failover only.
- **Database Mirroring (Legacy):** For one-to-one database replication. Replaced by Always On AGs.

## Always On Availability Groups
- **Primary vs. Secondary:** One primary (R/W), multiple secondaries (R/O or non-accessible).
- **Listeners:** A virtual network name that allows applications to connect without knowing which server is currently the primary.
- **Read-Only Routing:** Directs read-intent queries to secondary replicas to offload the primary.
- **Multi-Subnet AGs:** For geographically dispersed clusters. Requires `MultiSubnetFailover=True` in the connection string for fast failover.

## Windows Server Failover Clustering (WSFC)
- **Quorum:** A mechanism to prevent "split-brain" scenarios. Requires a majority of "votes" to remain active.
- **Witnesses:** A neutral "vote" (Disk, File Share, or Cloud Witness) to resolve ties in even-numbered clusters.

## Recovery Metrics (RPO/RTO)
- **RPO (Recovery Point Objective):** Maximum tolerable data loss.
- **RTO (Recovery Time Objective):** Maximum tolerable downtime.
- **HA Solutions** typically aim for RPO = 0 and RTO < 60 seconds.

## Best Practices (2024-2025)
- **Cloud Witness:** Use Azure Storage for a cost-effective and low-maintenance cluster witness.
- **Distributed Availability Groups:** For linking AGs across different clusters or data centers.
- **Automatic Seedings:** Simplifies the initialization of secondary replicas.
- **Performance Tuning:** Monitor `HADR_SYNC_COMMIT` wait stats to identify network/disk bottlenecks in synchronous replicas.

## Key T-SQL Commands
- `ALTER AVAILABILITY GROUP [AGName] FAILOVER;`
- `SELECT * FROM sys.dm_hadr_availability_replica_states;`
- `SELECT * FROM sys.dm_hadr_database_replica_cluster_states;`

## References
- [Microsoft Documentation: Always On Availability Groups](https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/always-on-availability-groups-sql-server)
- [Brent Ozar's Guide to Always On AGs](https://www.brentozar.com/sql/alwayson-availability-groups/)
- [SQL Server High Availability Checklist](https://www.mssqltips.com/sqlservertutorial/30/sql-server-high-availability-checklist/)
