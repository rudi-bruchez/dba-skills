---
name: sqlserver-high-availability
description: Configures and monitors SQL Server high availability using Always On Availability Groups, Failover Cluster Instances, and log shipping. Use when setting up HA/DR solutions, monitoring replication health, performing planned failovers, or troubleshooting synchronization lag.
metadata:
  author: dba-skills
  version: "1.0"
  platform: SQL Server 2016+
---

# SQL Server High Availability

## HA Technology Comparison

| Technology | RPO | RTO | Auto Failover | Read Scale-Out | Edition |
|---|---|---|---|---|---|
| Always On AG (synchronous) | 0 (zero data loss) | Seconds | Yes (with WSFC) | Yes (readable secondary) | Enterprise; Standard (2 nodes) |
| Always On AG (asynchronous) | Minutes | Minutes | No | Yes | Enterprise; Standard |
| Failover Cluster Instance (FCI) | 0 (shared storage) | 1–5 min | Yes | No | Enterprise; Standard |
| Log Shipping | Minutes (per interval) | Minutes–hours | No (manual) | Yes (warm standby) | All editions |
| Distributed AG (DAG) | Minutes (async) | Minutes | No | Yes | Enterprise |

**Selection guidance:**
- RPO = 0 / RTO < 30 sec → Synchronous AG with automatic failover
- RPO < 15 min / RTO < 30 min → Asynchronous AG or log shipping
- All databases must fail over together → FCI
- Geo-DR across two WSFC clusters → Distributed AG

---

## Always On AG Health Check Queries

Run these on the primary replica to assess the full AG health.

### Replica and synchronization state
```sql
SELECT
    ag.name                       AS ag_name,
    ar.replica_server_name,
    ar.availability_mode_desc,
    ar.failover_mode_desc,
    ars.role_desc                 AS current_role,
    ars.operational_state_desc,
    ars.connected_state_desc,
    ars.synchronization_health_desc,
    ars.last_connect_error_description
FROM sys.availability_groups ag
JOIN sys.availability_replicas ar
    ON ag.group_id = ar.group_id
JOIN sys.dm_hadr_availability_replica_states ars
    ON ar.replica_id = ars.replica_id
ORDER BY ag.name, ars.role_desc;
```

### Database synchronization detail
```sql
SELECT
    DB_NAME(drs.database_id)   AS database_name,
    ar.replica_server_name,
    drs.synchronization_state_desc,
    drs.synchronization_health_desc,
    drs.log_send_queue_size,   -- KB waiting to be sent (sync replica: should be ~0)
    drs.log_send_rate,         -- KB/sec currently sending
    drs.redo_queue_size,       -- KB waiting to be applied on secondary
    drs.redo_rate,             -- KB/sec being applied
    drs.last_commit_time,
    drs.last_redone_time,
    drs.last_received_time
FROM sys.dm_hadr_database_replica_states drs
JOIN sys.availability_replicas ar ON drs.replica_id = ar.replica_id
ORDER BY drs.database_id, ar.replica_server_name;
-- log_send_queue_size > 0 on a synchronous replica = synchronization problem
```

### AG listener and endpoint status
```sql
-- Listener
SELECT ag.name AS ag_name, al.dns_name AS listener_name,
       ali.ip_address, ali.ip_subnet_mask, al.port
FROM sys.availability_group_listeners al
JOIN sys.availability_groups ag ON al.group_id = ag.group_id
JOIN sys.availability_group_listener_ip_addresses ali
    ON al.listener_id = ali.listener_id;

-- Endpoint health (used by AG for log transport)
SELECT name, state_desc, role_desc, connection_auth_desc,
       encryption_algorithm_desc, port
FROM sys.database_mirroring_endpoints;
```

---

## Monitoring Replication Lag

### Estimated data loss for async replicas
```sql
SELECT
    DB_NAME(drs.database_id)   AS database_name,
    ar.replica_server_name,
    ar.availability_mode_desc,
    drs.log_send_queue_size    AS potential_data_loss_kb,
    drs.redo_queue_size        AS lag_to_apply_kb,
    DATEDIFF(SECOND,
        drs.last_commit_time,
        (SELECT MAX(last_commit_time)
         FROM sys.dm_hadr_database_replica_states
         WHERE database_id = drs.database_id AND is_local = 0))
        AS estimated_data_loss_sec
FROM sys.dm_hadr_database_replica_states drs
JOIN sys.availability_replicas ar ON drs.replica_id = ar.replica_id
WHERE ar.availability_mode = 0;  -- Async replicas only
-- estimated_data_loss_sec > 60 = investigate network or secondary I/O
```

### Alert: secondary not synchronizing
```sql
SELECT
    DB_NAME(drs.database_id) AS db_name,
    ar.replica_server_name,
    drs.synchronization_state_desc,
    drs.redo_queue_size
FROM sys.dm_hadr_database_replica_states drs
JOIN sys.availability_replicas ar ON drs.replica_id = ar.replica_id
WHERE drs.synchronization_state_desc NOT IN ('SYNCHRONIZED', 'SYNCHRONIZING')
  AND ar.replica_server_name <> @@SERVERNAME;
```

---

## Failover Decision Tree and Commands

```
Is the primary reachable?
  YES → Is the outage planned?
          YES → Perform planned manual failover (no data loss)
          NO  → Can you wait for sync?
                  YES → Wait for sync_state = SYNCHRONIZED, then planned failover
                  NO  → Force failover with data loss (disaster scenario only)
  NO  → WSFC has automatic failover configured?
          YES → Automatic failover occurs; verify after
          NO  → Connect to secondary replica console, force failover
```

### Planned manual failover (zero data loss — from target secondary)
```sql
-- Run on the replica you want to become the new primary
ALTER AVAILABILITY GROUP [AG_Production] FAILOVER;
-- Prerequisites: sync replica must be in SYNCHRONIZED state
-- Verify new primary: SELECT ars.role_desc FROM sys.dm_hadr_availability_replica_states ars
```

### Forced failover with potential data loss (disaster only — from new primary)
```sql
-- Use only when primary is completely unavailable and outage is unacceptable
ALTER AVAILABILITY GROUP [AG_Production] FORCE_FAILOVER_ALLOW_DATA_LOSS;

-- After forced failover: resume data movement on each database
ALTER DATABASE [AppDB1] SET HADR RESUME;
ALTER DATABASE [AppDB2] SET HADR RESUME;

-- Resynchronize the old primary when it comes back online
-- It will rejoin as a secondary and begin resynchronizing automatically
```

### Verify AG state after failover
```sql
SELECT ag.name, ars.role_desc, ars.operational_state_desc,
       ars.synchronization_health_desc, ars.connected_state_desc
FROM sys.availability_groups ag
JOIN sys.availability_replicas ar ON ag.group_id = ar.group_id
JOIN sys.dm_hadr_availability_replica_states ars ON ar.replica_id = ars.replica_id;
```

---

## Troubleshooting Sync Issues

### Common AG sync problems and fixes

| Symptom | Likely Cause | Fix |
|---|---|---|
| `synchronization_state = NOT SYNCHRONIZING` | Network failure, endpoint down, secondary restart | Check endpoint (`sys.database_mirroring_endpoints`); restart AG endpoint |
| `log_send_queue_size` growing on sync replica | Secondary too slow to apply log; network saturation | Check secondary I/O; reduce write workload on primary; consider async mode |
| `redo_queue_size` growing | Secondary redo thread cannot keep up | Check secondary disk I/O; check for long-running queries on secondary |
| AG health = NOT_HEALTHY | Database-level issue | Check `sys.dm_hadr_database_replica_states` for individual DB health |
| Listener not reachable | WSFC/network issue | Check WSFC cluster health; verify listener IP is online |

### Check AG endpoint connectivity
```sql
-- Verify endpoint is listening
SELECT name, state_desc, port
FROM sys.database_mirroring_endpoints;
-- state_desc must be STARTED. If STOPPED: ALTER ENDPOINT [Hadr_endpoint] STATE = STARTED;
```

### Check WSFC quorum (PowerShell on cluster node)
```powershell
Get-ClusterQuorum
Get-ClusterNode | Select-Object Name, State
# For two-node clusters: configure a cloud witness to avoid split-brain
Set-ClusterQuorum -CloudWitness -AccountName "storageaccount" -AccessKey "key=="
```

---

## Log Shipping Monitoring

### Check log shipping status on primary
```sql
SELECT
    primary_database,
    last_backup_date,
    last_backup_file,
    backup_threshold,
    DATEDIFF(MINUTE, last_backup_date, GETDATE()) AS minutes_since_backup,
    threshold_alert_enabled
FROM msdb.dbo.log_shipping_monitor_primary;
-- minutes_since_backup > backup_threshold = alert condition
```

### Check log shipping lag on secondary
```sql
SELECT
    secondary_database,
    last_restored_date,
    last_restored_file,
    restore_threshold,
    DATEDIFF(MINUTE, last_restored_date, GETDATE()) AS minutes_since_restore
FROM msdb.dbo.log_shipping_monitor_secondary;
-- minutes_since_restore > restore_threshold = log copy/restore job failing
```

### Log shipping history errors
```sql
SELECT TOP 50 *
FROM msdb.dbo.log_shipping_monitor_history_detail
WHERE error_number > 0
ORDER BY log_time DESC;
```

### Check log shipping jobs
```sql
SELECT j.name, j.enabled, jh.run_status, jh.run_date, jh.run_time, jh.message
FROM msdb.dbo.sysjobs j
JOIN msdb.dbo.sysjobhistory jh ON j.job_id = jh.job_id
WHERE j.name LIKE 'LSBackup%' OR j.name LIKE 'LSCopy%' OR j.name LIKE 'LSRestore%'
ORDER BY jh.run_date DESC, jh.run_time DESC;
-- run_status: 1 = success, 0 = failed
```

---

## FCI Health Check

```sql
-- Active node
SELECT SERVERPROPERTY('ComputerNamePhysicalNetBIOS') AS active_node;

-- All cluster nodes and their ownership
SELECT * FROM sys.dm_os_cluster_nodes;

-- Cluster properties
SELECT * FROM sys.dm_os_cluster_properties;
```

**FCI failover** is performed via Windows Server Failover Cluster Manager or PowerShell:
```powershell
# Move SQL Server FCI role to another node
Move-ClusterGroup -Name "SQL Server (MSSQLSERVER)" -Node "SQL02"
```

---

## Backup Strategy for Availability Groups

When databases are in an AG, run backups on the secondary to reduce primary load.

```sql
-- Use fn_hadr_backup_is_preferred_replica to run only on the preferred backup target
IF sys.fn_hadr_backup_is_preferred_replica('AppDB1') = 1
BEGIN
    BACKUP DATABASE [AppDB1]
    TO DISK = N'\\Backup\AppDB1_Full.bak'
    WITH COMPRESSION, CHECKSUM, COPY_ONLY;
    -- COPY_ONLY on secondary prevents breaking the differential base on the primary
END;
```

**Configure preferred backup target on the AG:**
```sql
ALTER AVAILABILITY GROUP [AG_Production]
MODIFY REPLICA ON N'SQL02' WITH (
    SECONDARY_ROLE (ALLOW_CONNECTIONS = READ_ONLY),
    AUTOMATED_BACKUP_PREFERENCE = SECONDARY
);
```

---

## RTO / RPO Design Guidance

| Tier | RPO | RTO | Recommended Solution |
|---|---|---|---|
| Mission Critical | 0 | < 30 sec | Synchronous AG with automatic failover; async replica at DR site |
| Business Critical | < 15 min | < 30 min | Asynchronous AG or log shipping (15-min interval); manual failover |
| Standard | < 24 hr | < 4 hr | Nightly full + hourly log backup; offsite backup copy |

---

## References

- [Always On reference](references/always-on-reference.md) — complete AG configuration and management commands
- [Failover procedures](references/failover-procedures.md) — step-by-step planned and forced failover runbooks
- [Monitoring queries](references/monitoring-queries.md) — all HA health monitoring queries
- [Examples](examples/examples.md) — AG health check, planned failover, troubleshooting sync lag, log shipping monitor
