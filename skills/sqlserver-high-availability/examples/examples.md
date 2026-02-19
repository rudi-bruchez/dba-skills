# SQL Server High Availability — Examples

## Example 1: AG Health Check — All Replicas Healthy

**Scenario:** Routine health check on a 3-node AG (SQL01 primary, SQL02 sync secondary, SQL03 async DR).

**Query output from replica and synchronization state query:**

| ag_name | replica_server_name | availability_mode_desc | failover_mode_desc | current_role | connected_state_desc | synchronization_health_desc |
|---|---|---|---|---|---|---|
| AG_Production | SQL01 | SYNCHRONOUS_COMMIT | AUTOMATIC | PRIMARY | CONNECTED | HEALTHY |
| AG_Production | SQL02 | SYNCHRONOUS_COMMIT | AUTOMATIC | SECONDARY | CONNECTED | HEALTHY |
| AG_Production | SQL03 | ASYNCHRONOUS_COMMIT | MANUAL | SECONDARY | CONNECTED | HEALTHY |

**Database synchronization detail:**

| database_name | replica_server_name | synchronization_state_desc | log_send_queue_kb | redo_queue_kb |
|---|---|---|---|---|
| AppDB1 | SQL02 | SYNCHRONIZED | 0 | 0 |
| AppDB1 | SQL03 | SYNCHRONIZING | 512 | 1024 |

**Interpretation:**
- SQL02 is fully synchronized (RPO = 0 for planned failover).
- SQL03 shows 512 KB in the send queue and 1 MB redo queue — normal for an async DR replica with active workload. Monitor; alert if `redo_queue_size` exceeds 50 MB.

---

## Example 2: Planned Failover — Maintenance Window

**Scenario:** Patching SQL01 (primary). Failing over to SQL02 during a scheduled maintenance window.

**Step 1 — Pre-failover check on SQL01:**
```sql
SELECT ar.replica_server_name, drs.synchronization_state_desc
FROM sys.availability_replicas ar
JOIN sys.dm_hadr_database_replica_states drs ON ar.replica_id = drs.replica_id
WHERE ar.replica_server_name = 'SQL02';
-- Result: synchronization_state_desc = SYNCHRONIZED  ← safe to fail over
```

**Step 2 — Execute on SQL02 (target):**
```sql
ALTER AVAILABILITY GROUP [AG_Production] FAILOVER;
```

**Step 3 — Verify on SQL02 (now primary):**
```sql
SELECT ar.replica_server_name, ars.role_desc, ars.synchronization_health_desc
FROM sys.availability_replicas ar
JOIN sys.dm_hadr_availability_replica_states ars ON ar.replica_id = ars.replica_id;
-- SQL02: PRIMARY, SQL01: SECONDARY
```

**Result:** Zero data loss. Application connections via `AG_Listener` automatically route to SQL02. After patching SQL01, perform failback using the same `ALTER AVAILABILITY GROUP FAILOVER` command executed on SQL01.

---

## Example 3: Troubleshooting Synchronization Lag

**Scenario:** Monitoring alert fires — `log_send_queue_size` on SQL02 is 45 MB (sync replica, should be ~0).

**Diagnosis:**

```sql
-- Step 1: Check which databases have large queues
SELECT
    DB_NAME(drs.database_id)    AS database_name,
    ar.replica_server_name,
    drs.synchronization_state_desc,
    drs.log_send_queue_size     AS send_queue_kb,
    drs.log_send_rate           AS send_rate_kbsec,
    drs.redo_queue_size         AS redo_queue_kb,
    drs.redo_rate               AS redo_rate_kbsec
FROM sys.dm_hadr_database_replica_states drs
JOIN sys.availability_replicas ar ON drs.replica_id = ar.replica_id
WHERE ar.replica_server_name = 'SQL02';
```

**Output:**

| database_name | synchronization_state_desc | send_queue_kb | send_rate_kbsec | redo_queue_kb | redo_rate_kbsec |
|---|---|---|---|---|---|
| AppDB1 | SYNCHRONIZING | 45056 | 8192 | 120000 | 4096 |

**Finding:** `redo_queue_size` is 120 MB and `redo_rate` is only 4 MB/sec — secondary redo thread is falling behind (48 sec at current rate). `log_send_rate` is 8 MB/sec (primary is generating more log than secondary can apply).

**Investigation on SQL02:**
```sql
-- Check SQL02 disk I/O latency
SELECT
    io_stall_read_ms / NULLIF(num_of_reads, 0)   AS avg_read_ms,
    io_stall_write_ms / NULLIF(num_of_writes, 0) AS avg_write_ms,
    physical_name
FROM sys.dm_io_virtual_file_stats(DB_ID('AppDB1'), NULL) vfs
JOIN sys.master_files mf ON vfs.database_id = mf.database_id AND vfs.file_id = mf.file_id;
-- High avg_write_ms (> 15 ms) indicates I/O bottleneck on secondary
```

**Resolution options:**
1. Move the secondary data files to faster storage (SSD).
2. Reduce write workload on primary during peak periods.
3. If the secondary cannot keep up consistently, consider converting it to `ASYNCHRONOUS_COMMIT` to remove the commit-wait dependency on primary.

---

## Example 4: Log Shipping Restore Lag Alert

**Scenario:** Log shipping restore job on secondary has not run in 2 hours; alert threshold is 90 minutes.

**Diagnosis:**

```sql
-- On secondary
SELECT
    secondary_database,
    last_restored_date,
    DATEDIFF(MINUTE, last_restored_date, GETDATE()) AS minutes_since_restore,
    restore_threshold
FROM msdb.dbo.log_shipping_monitor_secondary;
-- Result: minutes_since_restore = 120, restore_threshold = 90
```

**Check job history for errors:**
```sql
SELECT TOP 10 *
FROM msdb.dbo.log_shipping_monitor_history_detail
WHERE agent_type = 2 AND error_number > 0  -- 2 = restore agent
ORDER BY log_time DESC;
-- Error: "Operating system error 3(The system cannot find the path specified.)"
```

**Finding:** The log copy destination path `D:\LogCopy` is inaccessible (disk failure). The copy job is failing, so there are no files to restore.

**Resolution:**
1. Fix the disk issue or redirect the copy destination path.
2. Update the log shipping copy job to point to the new path:
```sql
EXEC msdb.dbo.sp_change_log_shipping_secondary_primary
    @primary_server = 'SQL01',
    @primary_database = 'PrimaryDB',
    @backup_source_directory = '\\SQL01\LogBackups',
    @backup_destination_directory = 'E:\LogCopy';  -- new path
```
3. Manually copy the missing log backup files and restore them with NORECOVERY.

---

## Example 5: FCI Node Failover Verification

**Scenario:** SQL Server FCI unexpectedly failed over from SQL01 to SQL02. Verifying the cluster is healthy.

```sql
-- Confirm active node
SELECT SERVERPROPERTY('ComputerNamePhysicalNetBIOS') AS active_node;
-- Result: SQL02

-- Check cluster node states
SELECT * FROM sys.dm_os_cluster_nodes;
-- node_name: SQL01, is_current_owner: 0, status_description: Up (or check if Paused/Down)
-- node_name: SQL02, is_current_owner: 1, status_description: Up
```

```powershell
# Check why failover occurred (Windows event log)
Get-WinEvent -LogName System | Where-Object { $_.Id -in 1069, 1205 } |
    Select-Object TimeCreated, Id, Message | Format-List
```

**Findings:** Event ID 1069 (cluster resource failed) on SQL01 due to disk timeout. The FCI automatically moved to SQL02.

**Resolution:** Investigate SQL01 disk I/O issues, fix underlying storage problem, then optionally move the group back:
```powershell
Move-ClusterGroup -Name "SQL Server (MSSQLSERVER)" -Node "SQL01"
```
