# Failover Procedures

A step-by-step guide to performing planned and unplanned failovers in an Always On Availability Group (AG) cluster.

## 1. Planned Failover (Manual)
Perform a planned failover to verify the cluster's readiness and ensure no data loss.
```sql
ALTER AVAILABILITY GROUP [AGName] FAILOVER;
```
Verify the failover status on both the old primary and new primary.

## 2. Unplanned Failover (Automatic)
In an automatic failover scenario:
1.  **Stop the primary instance or disconnect it from the network.**
2.  **Verify that the standby server has been promoted to the new primary.**
3.  **Check for data loss if using asynchronous replication.**
4.  **Re-configure the old primary as a standby after fixing the issue.**

## 3. Resolving "Split-Brain" Scenarios
A "split-brain" scenario occurs when two nodes in the cluster think they are the primary.
1.  **Identify the true primary node.**
2.  **Stop the other node.**
3.  **Force the cluster back to its correct state using the `FORCE_FAILOVER_ALLOW_DATA_LOSS` option.**
```sql
ALTER AVAILABILITY GROUP [AGName] FORCE_FAILOVER_ALLOW_DATA_LOSS;
```

## 4. Multi-Subnet AGs
Ensure `MultiSubnetFailover=True` is set in connection strings for fast failover.

## 5. Automated Failover (Patroni)
If using Patroni, the failover process is automated.

## References
- [Microsoft Documentation: Always On Availability Groups - Failover](https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/failover-and-failover-modes-always-on-availability-groups)
- [Brent Ozar's Guide to Always On AG Failover](https://www.brentozar.com/sql/alwayson-availability-groups-failover/)
- [pganalyze: Guide to PostgreSQL Failover](https://pganalyze.com/blog/postgresql-failover)
---
