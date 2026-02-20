---
name: sqlserver-high-availability
description: Expert skill for managing SQL Server High Availability (HA) and Disaster Recovery (DR) solutions, focusing on Always On Availability Groups.
version: 1.0.0
tags:
  - sqlserver
  - dba
  - high-availability
  - always-on
  - disaster-recovery
---

# SQL Server High Availability (HA) & Disaster Recovery (DR)

This skill provides expert guidance for designing, implementing, and managing SQL Server HA and DR solutions, including Always On Availability Groups (AGs) and Failover Cluster Instances (FCIs).

## Core Capabilities

- **Always On Availability Groups (AGs):** Setting up primary and secondary replicas, listeners, and read-only routing.
- **Failover Cluster Instances (FCIs):** Implementing instance-level HA using Windows Server Failover Clustering (WSFC).
- **Log Shipping:** Setting up database-level DR for asynchronous replication.
- **Monitoring HA/DR Health:** Tracking replica status, synchronization, and failover readiness.
- **Failover Procedures:** Performing planned and unplanned failovers and resolving "split-brain" scenarios.

## Workflow: Always On AG Setup

1.  **Prepare the Environment:**
    - Configure Windows Server Failover Clustering (WSFC).
    - Enable Always On Availability Groups on each SQL Server instance.
2.  **Create the Availability Group:**
    - Choose the databases to include.
    - Specify the primary and secondary replicas.
    - Select the availability mode (Synchronous or Asynchronous).
3.  **Configure the Listener:** Create a virtual network name for application connections.
4.  **Implement Read-Only Routing:** Direct read-intent queries to secondary replicas to offload the primary.
5.  **Monitor Health:** Use `sys.dm_hadr_availability_replica_states` and `sys.dm_hadr_database_replica_cluster_states`.
6.  **Test Failover:** Regularly perform planned failovers to verify the cluster's readiness.

## Essential Commands

### 1. Check Availability Group Status
```sql
SELECT
    ag.name AS AGName,
    ar.replica_server_name AS ReplicaName,
    ar.availability_mode_desc,
    rs.role_desc,
    rs.operational_state_desc,
    rs.connected_state_desc,
    rs.recovery_health_desc,
    rs.synchronization_health_desc
FROM sys.availability_groups AS ag
JOIN sys.availability_replicas AS ar ON ag.group_id = ar.group_id
JOIN sys.dm_hadr_availability_replica_states AS rs ON ar.replica_id = rs.replica_id;
```

### 2. Check Database Replication Status
```sql
SELECT
    ag.name AS AGName,
    ar.replica_server_name AS ReplicaName,
    drcs.database_name,
    drcs.synchronization_state_desc,
    drcs.is_failover_ready
FROM sys.availability_groups AS ag
JOIN sys.availability_replicas AS ar ON ag.group_id = ar.group_id
JOIN sys.dm_hadr_database_replica_cluster_states AS drcs ON ar.replica_id = drcs.replica_id;
```

### 3. Perform a Planned Failover
```sql
ALTER AVAILABILITY GROUP [AGName] FAILOVER;
```

### 4. Create a Listener
```sql
ALTER AVAILABILITY GROUP [AGName]
ADD LISTENER 'AGListener' (
    WITH IP (('10.0.0.100', '255.255.255.0')),
    PORT = 1433
);
```

## Best Practices (2024-2025)

- **Cloud Witness:** Use Azure Storage for a cost-effective and low-maintenance cluster witness.
- **Distributed Availability Groups:** For linking AGs across different clusters or data centers.
- **Automatic Seedings:** Simplifies the initialization of secondary replicas.
- **Performance Tuning:** Monitor `HADR_SYNC_COMMIT` wait stats to identify network/disk bottlenecks in synchronous replicas.
- **Multi-Subnet AGs:** Ensure `MultiSubnetFailover=True` is set in connection strings for fast failover.

## References

- [Always On Reference](./references/always-on-reference.md)
- [Failover Procedures](./references/failover-procedures.md)
- [Monitoring Queries](./references/monitoring-queries.md)
---
