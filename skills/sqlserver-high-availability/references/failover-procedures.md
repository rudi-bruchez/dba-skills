# Failover Procedures — Step-by-Step Runbooks

## Procedure 1: Planned Manual Failover (Always On AG)

**When to use:** Scheduled maintenance, patching, testing. No data loss.
**Prerequisite:** Target secondary must be in `SYNCHRONOUS_COMMIT` mode and `SYNCHRONIZED` state.

### Pre-failover checklist

```sql
-- 1. Confirm the target secondary is synchronized
SELECT
    ar.replica_server_name,
    ars.role_desc,
    ars.synchronization_health_desc,
    drs.synchronization_state_desc
FROM sys.availability_replicas ar
JOIN sys.dm_hadr_availability_replica_states ars ON ar.replica_id = ars.replica_id
JOIN sys.dm_hadr_database_replica_states drs ON ar.replica_id = drs.replica_id
WHERE ar.replica_server_name = N'SQL02';  -- Target secondary
-- synchronization_state_desc must be SYNCHRONIZED for all databases

-- 2. Check no long-running transactions on primary that will delay failover
SELECT session_id, blocking_session_id, wait_type, total_elapsed_time / 1000 AS elapsed_sec,
       DB_NAME(database_id) AS db_name, [text]
FROM sys.dm_exec_requests
CROSS APPLY sys.dm_exec_sql_text(sql_handle)
WHERE total_elapsed_time > 30000  -- > 30 seconds
ORDER BY total_elapsed_time DESC;

-- 3. Identify active application connections on primary
SELECT login_name, host_name, COUNT(*) AS connection_count
FROM sys.dm_exec_sessions
WHERE is_user_process = 1
GROUP BY login_name, host_name
ORDER BY connection_count DESC;
```

### Execute planned failover

```sql
-- Run on the TARGET SECONDARY (SQL02), NOT on the current primary
-- This initiates a graceful role swap with zero data loss
ALTER AVAILABILITY GROUP [AG_Production] FAILOVER;
```

### Post-failover verification

```sql
-- Run on the new primary (SQL02) to confirm roles
SELECT
    ag.name AS ag_name,
    ar.replica_server_name,
    ars.role_desc,
    ars.operational_state_desc,
    ars.synchronization_health_desc,
    ars.connected_state_desc
FROM sys.availability_groups ag
JOIN sys.availability_replicas ar ON ag.group_id = ar.group_id
JOIN sys.dm_hadr_availability_replica_states ars ON ar.replica_id = ars.replica_id
ORDER BY ars.role_desc;

-- Confirm listener is routing to new primary
SELECT
    al.dns_name AS listener,
    ags.primary_replica,
    ags.synchronization_health_desc
FROM sys.availability_group_listeners al
JOIN sys.dm_hadr_availability_group_states ags ON al.group_id = ags.group_id;
```

---

## Procedure 2: Forced Failover with Potential Data Loss

**When to use:** Primary is completely unreachable and outage duration is unacceptable.
**Risk:** Transactions committed on the primary but not yet hardened on the secondary will be lost.

### Decision criteria before forcing

1. Primary cannot be reached by any means (RDP, network, physical access).
2. WSFC automatic failover did NOT occur (no configured auto-failover, or quorum lost).
3. Data loss is acceptable or unavoidable given the outage.
4. The DR/RTO target cannot be met by waiting for primary recovery.

### Execute forced failover

```sql
-- Run on the SECONDARY that will become the new primary (e.g., SQL02)
ALTER AVAILABILITY GROUP [AG_Production] FORCE_FAILOVER_ALLOW_DATA_LOSS;

-- After forced failover: resume data movement on each AG database
-- (databases may enter NOT SYNCHRONIZING state after forced failover)
ALTER DATABASE [AppDB1] SET HADR RESUME;
ALTER DATABASE [AppDB2] SET HADR RESUME;
```

### Post-forced-failover steps

```sql
-- 1. Verify databases are accessible on the new primary
SELECT name, state_desc, recovery_model_desc
FROM sys.databases
WHERE name IN ('AppDB1', 'AppDB2');

-- 2. Check for any databases still in RESTORING or RECOVERY PENDING
SELECT name, state_desc FROM sys.databases WHERE state_desc != 'ONLINE';

-- 3. Monitor resynchronization once old primary is restored
SELECT
    DB_NAME(drs.database_id) AS database_name,
    ar.replica_server_name,
    drs.synchronization_state_desc,
    drs.log_send_queue_size,
    drs.redo_queue_size
FROM sys.dm_hadr_database_replica_states drs
JOIN sys.availability_replicas ar ON drs.replica_id = ar.replica_id
ORDER BY drs.database_id;

-- 4. After old primary is recovered and rejoined as secondary,
--    consider performing a planned failover back to restore original roles
```

---

## Procedure 3: Log Shipping Failover (Manual)

**When to use:** Primary server fails; log shipping secondary must take over.

### Verify secondary readiness

```sql
-- On secondary: check most recent restore
SELECT
    secondary_database,
    last_restored_date,
    last_restored_file,
    restore_threshold,
    DATEDIFF(MINUTE, last_restored_date, GETDATE()) AS minutes_since_restore
FROM msdb.dbo.log_shipping_monitor_secondary;
```

### Restore remaining log backups and bring online

```sql
-- 1. Copy and restore any remaining log backup files not yet applied
--    (restore each in order with NORECOVERY until the last one)
RESTORE LOG [SecondaryDB]
FROM DISK = N'\\SQL01\LogBackups\PrimaryDB_20240115_2359.trn'
WITH NORECOVERY;

-- 2. Bring the secondary database online (WITH RECOVERY ends the restore chain)
RESTORE DATABASE [SecondaryDB] WITH RECOVERY;

-- 3. Update application connection strings to point to the secondary server
--    (no listener exists for log shipping — requires DNS or connection string change)
```

### Remove log shipping configuration after failover

```sql
-- On old secondary (now primary): remove log shipping secondary config
EXEC master.dbo.sp_delete_log_shipping_secondary_database
    @secondary_database = N'SecondaryDB',
    @primary_server = N'SQL01',
    @primary_database = N'PrimaryDB';
```

---

## Procedure 4: FCI Failover

**When to use:** Active cluster node fails; SQL Server must move to another node.

### Manual FCI failover (PowerShell)

```powershell
# List current cluster group owners
Get-ClusterGroup | Where-Object { $_.Name -like "*SQL*" } | Select-Object Name, OwnerNode, State

# Move the SQL Server cluster group to another node
Move-ClusterGroup -Name "SQL Server (MSSQLSERVER)" -Node "SQL02"

# Verify the move succeeded
Get-ClusterGroup -Name "SQL Server (MSSQLSERVER)" | Select-Object Name, OwnerNode, State
```

```sql
-- Verify SQL Server is running on expected node after failover
SELECT SERVERPROPERTY('ComputerNamePhysicalNetBIOS') AS active_node;
SELECT * FROM sys.dm_os_cluster_nodes;
```

---

## Procedure 5: Failback to Original Primary

After an unplanned failover or forced failover, restore normal roles.

```sql
-- 1. Ensure the old primary has been recovered and is now a synchronized secondary
SELECT ar.replica_server_name, ars.role_desc, drs.synchronization_state_desc
FROM sys.availability_replicas ar
JOIN sys.dm_hadr_availability_replica_states ars ON ar.replica_id = ars.replica_id
JOIN sys.dm_hadr_database_replica_states drs ON ar.replica_id = drs.replica_id
WHERE ar.replica_server_name = N'SQL01';
-- synchronization_state_desc must be SYNCHRONIZED before failback

-- 2. Perform planned failover back to SQL01 (run on SQL01 — target of failback)
ALTER AVAILABILITY GROUP [AG_Production] FAILOVER;

-- 3. Verify SQL01 is now primary
SELECT ar.replica_server_name, ars.role_desc
FROM sys.availability_replicas ar
JOIN sys.dm_hadr_availability_replica_states ars ON ar.replica_id = ars.replica_id;
```

---

## Health Check Thresholds

| Metric | Healthy | Warning | Critical |
|---|---|---|---|
| `synchronization_state_desc` | SYNCHRONIZED | SYNCHRONIZING | NOT SYNCHRONIZING |
| `log_send_queue_size` (sync replica) | < 100 KB | 100 KB – 1 MB | > 1 MB |
| `redo_queue_size` | < 1 MB | 1 – 50 MB | > 50 MB |
| `estimated_data_loss_sec` (async) | < 30 sec | 30 – 120 sec | > 120 sec |
| `connected_state_desc` | CONNECTED | — | DISCONNECTED |
| `operational_state_desc` | ONLINE | — | OFFLINE / FAILED |
