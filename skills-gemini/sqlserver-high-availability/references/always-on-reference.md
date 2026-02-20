# Always On Reference

A guide to the physical and logical structure of SQL Server Always On Availability Groups (AGs).

## 1. Primary vs. Secondary Replicas
One primary (R/W), multiple secondaries (R/O or non-accessible).
- **Synchronous Replicas:** The primary waits for the secondary to confirm receipt and write of WAL before committing. Guarantees no data loss but can increase latency.
- **Asynchronous Replicas:** The primary commits immediately without waiting for the secondary. Low performance impact, but small risk of data loss on primary failure.

## 2. Listeners
A virtual network name that allows applications to connect without knowing which server is currently the primary.
- **Pros:** Simplifies application configuration and provides fast failover.

## 3. Read-Only Routing
Directs read-intent queries to secondary replicas to offload the primary.
- **Setup:** Configure the `read_only_routing_url` and `read_only_routing_list` for each replica.

## 4. Multi-Subnet AGs
For geographically dispersed clusters. Requires `MultiSubnetFailover=True` in the connection string for fast failover.

## 5. Quorum and Witnesses
A mechanism to prevent "split-brain" scenarios. Requires a majority of "votes" to remain active.
- **Witnesses:** A neutral "vote" (Disk, File Share, or Cloud Witness) to resolve ties in even-numbered clusters.

## References
- [Microsoft Documentation: Always On Availability Groups](https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/always-on-availability-groups-sql-server)
- [Brent Ozar's Guide to Always On AGs](https://www.brentozar.com/sql/alwayson-availability-groups/)
- [SQL Server High Availability Checklist](https://www.mssqltips.com/sqlservertutorial/30/sql-server-high-availability-checklist/)
---
