# HA Monitoring Queries — Reference

## Always On AG — Complete Health Dashboard

```sql
-- Full AG health: one row per replica per database
SELECT
    ag.name                             AS ag_name,
    ar.replica_server_name,
    ar.availability_mode_desc,
    ar.failover_mode_desc,
    ars.role_desc                       AS replica_role,
    ars.operational_state_desc,
    ars.connected_state_desc,
    ars.synchronization_health_desc     AS replica_sync_health,
    ars.last_connect_error_description,
    DB_NAME(drs.database_id)            AS database_name,
    drs.synchronization_state_desc      AS db_sync_state,
    drs.synchronization_health_desc     AS db_sync_health,
    drs.log_send_queue_size             AS log_send_queue_kb,
    drs.log_send_rate                   AS log_send_rate_kbsec,
    drs.redo_queue_size                 AS redo_queue_kb,
    drs.redo_rate                       AS redo_rate_kbsec,
    drs.last_commit_time,
    drs.last_redone_time,
    drs.last_received_time
FROM sys.availability_groups ag
JOIN sys.availability_replicas ar
    ON ag.group_id = ar.group_id
JOIN sys.dm_hadr_availability_replica_states ars
    ON ar.replica_id = ars.replica_id
JOIN sys.dm_hadr_database_replica_states drs
    ON ar.replica_id = drs.replica_id
ORDER BY ag.name, ars.role_desc DESC, DB_NAME(drs.database_id);
```

---

## AG-Level State Summary

```sql
-- Summarized: one row per AG
SELECT
    ags.group_id,
    ag.name                             AS ag_name,
    ags.primary_replica,
    ags.primary_recovery_health_desc,
    ags.secondary_recovery_health_desc,
    ags.synchronization_health_desc
FROM sys.dm_hadr_availability_group_states ags
JOIN sys.availability_groups ag ON ags.group_id = ag.group_id;
```

---

## Replica Lag and Estimated Data Loss

```sql
-- Estimated data loss for async replicas (run on primary)
SELECT
    DB_NAME(drs.database_id)            AS database_name,
    ar.replica_server_name,
    ar.availability_mode_desc,
    drs.log_send_queue_size             AS potential_data_loss_kb,
    drs.redo_queue_size                 AS lag_to_apply_kb,
    DATEDIFF(SECOND,
        drs.last_commit_time,
        (SELECT MAX(last_commit_time)
         FROM sys.dm_hadr_database_replica_states
         WHERE database_id = drs.database_id AND is_local = 0))
        AS estimated_data_loss_sec
FROM sys.dm_hadr_database_replica_states drs
JOIN sys.availability_replicas ar ON drs.replica_id = ar.replica_id
WHERE ar.availability_mode = 0  -- Async replicas only
ORDER BY estimated_data_loss_sec DESC;
```

---

## Alert: Replicas Not Synchronizing

```sql
-- Returns rows only when a problem exists (suitable for SQL Agent alert job)
SELECT
    DB_NAME(drs.database_id)        AS db_name,
    ar.replica_server_name,
    drs.synchronization_state_desc,
    drs.synchronization_health_desc,
    drs.redo_queue_size,
    ars.last_connect_error_description
FROM sys.dm_hadr_database_replica_states drs
JOIN sys.availability_replicas ar
    ON drs.replica_id = ar.replica_id
JOIN sys.dm_hadr_availability_replica_states ars
    ON ar.replica_id = ars.replica_id
WHERE drs.synchronization_state_desc NOT IN ('SYNCHRONIZED', 'SYNCHRONIZING')
  AND ar.replica_server_name <> @@SERVERNAME;
```

---

## Listener and Endpoint Health

```sql
-- Listener configuration and IP addresses
SELECT
    ag.name                         AS ag_name,
    al.dns_name                     AS listener_name,
    ali.ip_address,
    ali.ip_subnet_mask,
    al.port,
    al.ip_configuration_string_from_cluster
FROM sys.availability_group_listeners al
JOIN sys.availability_groups ag ON al.group_id = ag.group_id
JOIN sys.availability_group_listener_ip_addresses ali ON al.listener_id = ali.listener_id;

-- Mirroring endpoint status (used for AG transport)
SELECT
    name,
    state_desc,
    role_desc,
    connection_auth_desc,
    encryption_algorithm_desc,
    port
FROM sys.database_mirroring_endpoints;
-- state_desc must be STARTED; if STOPPED: ALTER ENDPOINT [Hadr_endpoint] STATE = STARTED;
```

---

## Log Shipping — Primary Monitoring

```sql
-- Primary server: backup lag
SELECT
    primary_database,
    last_backup_date,
    last_backup_file,
    backup_threshold,
    DATEDIFF(MINUTE, last_backup_date, GETDATE()) AS minutes_since_backup,
    threshold_alert_enabled
FROM msdb.dbo.log_shipping_monitor_primary
ORDER BY DATEDIFF(MINUTE, last_backup_date, GETDATE()) DESC;
-- Alert when minutes_since_backup > backup_threshold

-- Recent backup history
SELECT TOP 20 *
FROM msdb.dbo.log_shipping_monitor_history_detail
WHERE agent_type = 0  -- 0 = backup agent
ORDER BY log_time DESC;
```

---

## Log Shipping — Secondary Monitoring

```sql
-- Secondary server: restore lag
SELECT
    secondary_database,
    last_restored_date,
    last_restored_file,
    restore_threshold,
    DATEDIFF(MINUTE, last_restored_date, GETDATE()) AS minutes_since_restore,
    restore_delay
FROM msdb.dbo.log_shipping_monitor_secondary
ORDER BY DATEDIFF(MINUTE, last_restored_date, GETDATE()) DESC;
-- Alert when minutes_since_restore > restore_threshold

-- Recent restore errors
SELECT TOP 50 *
FROM msdb.dbo.log_shipping_monitor_history_detail
WHERE error_number > 0
ORDER BY log_time DESC;

-- Log shipping SQL Agent jobs status
SELECT
    j.name,
    j.enabled,
    jh.run_status,         -- 1 = success, 0 = failed
    jh.run_date,
    jh.run_time,
    jh.message
FROM msdb.dbo.sysjobs j
JOIN msdb.dbo.sysjobhistory jh ON j.job_id = jh.job_id
WHERE j.name LIKE 'LSBackup%' OR j.name LIKE 'LSCopy%' OR j.name LIKE 'LSRestore%'
ORDER BY jh.run_date DESC, jh.run_time DESC;
```

---

## FCI — Cluster Node Health

```sql
-- Active node and cluster properties
SELECT SERVERPROPERTY('ComputerNamePhysicalNetBIOS') AS active_node;
SELECT * FROM sys.dm_os_cluster_nodes;
SELECT * FROM sys.dm_os_cluster_properties;
```

```powershell
# PowerShell: full cluster node health
Get-ClusterNode | Select-Object Name, State, NodeWeight
Get-ClusterGroup | Where-Object { $_.Name -like "*SQL*" } | Select-Object Name, OwnerNode, State
```

---

## AG Automatic Seeding Progress

```sql
-- Monitor automatic seeding (SQL 2016+)
SELECT
    start_time,
    completion_time,
    is_master,
    current_state,
    failure_message,
    database_name,
    ROUND(transferred_size_bytes / 1048576.0, 1)    AS transferred_mb,
    ROUND(database_size_bytes / 1048576.0, 1)       AS database_size_mb,
    ROUND(transferred_size_bytes * 100.0
          / NULLIF(database_size_bytes, 0), 1)      AS pct_complete
FROM sys.dm_hadr_automatic_seeding;
```

---

## Monitoring Thresholds Reference

| Metric | Source | Warning | Critical |
|---|---|---|---|
| Minutes since last LS backup | `log_shipping_monitor_primary` | > backup_threshold | > backup_threshold × 2 |
| Minutes since last LS restore | `log_shipping_monitor_secondary` | > restore_threshold | > restore_threshold × 2 |
| `log_send_queue_size` (sync replica) | `dm_hadr_database_replica_states` | > 100 KB | > 1 MB |
| `redo_queue_size` | `dm_hadr_database_replica_states` | > 1 MB | > 50 MB |
| `estimated_data_loss_sec` (async) | `dm_hadr_database_replica_states` | > 30 sec | > 120 sec |
| Replica sync health | `dm_hadr_availability_replica_states` | PARTIALLY_HEALTHY | NOT_HEALTHY |
| Endpoint state | `database_mirroring_endpoints` | — | STOPPED |
