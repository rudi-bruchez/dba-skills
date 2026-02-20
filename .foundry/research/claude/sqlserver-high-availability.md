# SQL Server High Availability & Disaster Recovery

## Overview

SQL Server provides multiple HA/DR technologies with different RTO, RPO, and complexity tradeoffs. Modern environments typically use Always On Availability Groups as the primary HA/DR solution. Understanding each technology's characteristics is essential for design decisions.

---

## HA/DR Technology Comparison

| Technology | RPO | RTO | Automatic Failover | Read Scale-Out | SQL Edition |
|---|---|---|---|---|---|
| Always On AG (sync) | 0 (zero data loss) | Seconds | Yes (with WSFC) | Yes (readable secondary) | Enterprise, Standard (2 nodes) |
| Always On AG (async) | Minutes | Minutes | No | Yes | Enterprise, Standard |
| Failover Cluster Instance (FCI) | 0 (shared storage) | 1-5 min | Yes | No (one active node) | Enterprise, Standard |
| Log Shipping | Minutes (interval) | Minutes-hours | No (manual) | Yes (warm standby) | All editions |
| Database Mirroring (deprecated) | Seconds | Seconds-minutes | Yes (with witness) | No | Deprecated in 2016 |
| Replication | Near real-time | Depends on replication type | No | Yes | All editions |
| Azure SQL Auto-failover Groups | 0 (sync within region) | Seconds | Yes | Yes | Azure SQL only |

---

## Always On Availability Groups (Primary HA/DR Solution)

### Architecture
- **AG**: Group of databases that fail over together as a unit.
- **Primary replica**: Accepts read/write connections; transactions hardened to log.
- **Secondary replicas**: Receive and apply log from primary. Up to 8 secondary replicas (5 synchronous in SQL 2016+).
- **Synchronous mode**: Primary waits for secondary to harden log before committing. RPO = 0.
- **Asynchronous mode**: Primary does not wait. Low overhead; potential data loss on failover.
- **Listener**: Virtual network name + IP that clients connect to. Routes to primary or readable secondary.
- **WSFC**: Windows Server Failover Cluster provides health detection and automatic failover infrastructure.
- **Quorum**: WSFC requires majority quorum (nodes + optionally file share/cloud witness).

### AG Prerequisites
```sql
-- Check if AG endpoint exists
SELECT * FROM sys.database_mirroring_endpoints;

-- Check if HADR is enabled on instance
SELECT SERVERPROPERTY('IsHadrEnabled') AS hadr_enabled;

-- Enable HADR (requires service restart)
-- Done via SQL Server Configuration Manager or PowerShell:
-- Enable-SqlAlwaysOn -ServerInstance "SQL01" -Force
```

### Create an Availability Group
```sql
-- On the primary replica
CREATE AVAILABILITY GROUP [AG_Production]
WITH (
    AUTOMATED_BACKUP_PREFERENCE = SECONDARY,  -- Prefer backups on secondary
    FAILURE_CONDITION_LEVEL = 3,              -- 1-5; 3 = default (server down, unresponsive)
    HEALTH_CHECK_TIMEOUT = 30000,             -- ms before health check triggers failover
    DB_FAILOVER = ON,                         -- SQL 2016+: database health triggers failover
    DTC_SUPPORT = NONE                        -- Distributed transactions support
)
FOR DATABASE [AppDB1], [AppDB2]
REPLICA ON
    N'SQL01' WITH (
        ENDPOINT_URL = N'TCP://SQL01.domain.com:5022',
        AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
        FAILOVER_MODE = AUTOMATIC,
        SEEDING_MODE = AUTOMATIC,             -- Auto seeding (SQL 2016+)
        SECONDARY_ROLE (ALLOW_CONNECTIONS = NO)
    ),
    N'SQL02' WITH (
        ENDPOINT_URL = N'TCP://SQL02.domain.com:5022',
        AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
        FAILOVER_MODE = AUTOMATIC,
        SEEDING_MODE = AUTOMATIC,
        SECONDARY_ROLE (ALLOW_CONNECTIONS = READ_ONLY)
    ),
    N'SQL03' WITH (  -- DR site - async
        ENDPOINT_URL = N'TCP://SQL03.domain.com:5022',
        AVAILABILITY_MODE = ASYNCHRONOUS_COMMIT,
        FAILOVER_MODE = MANUAL,
        SEEDING_MODE = AUTOMATIC,
        SECONDARY_ROLE (ALLOW_CONNECTIONS = READ_ONLY)
    );

-- Create listener
ALTER AVAILABILITY GROUP [AG_Production]
ADD LISTENER N'AG_Listener' (
    WITH IP ((N'10.0.1.50', N'255.255.255.0')),
    PORT = 1433
);
```

### Join Secondary Replica
```sql
-- On each secondary replica
ALTER AVAILABILITY GROUP [AG_Production] JOIN;
ALTER AVAILABILITY GROUP [AG_Production] GRANT CREATE ANY DATABASE;
-- With automatic seeding, databases will seed automatically
```

### Failover Commands
```sql
-- Planned manual failover (from new primary; graceful, no data loss)
ALTER AVAILABILITY GROUP [AG_Production] FAILOVER;

-- Forced failover with potential data loss (from new primary; use in disaster only)
ALTER AVAILABILITY GROUP [AG_Production] FORCE_FAILOVER_ALLOW_DATA_LOSS;

-- After forced failover: resume data movement and resynchronize
ALTER DATABASE [AppDB1] SET HADR RESUME;
ALTER DATABASE [AppDB2] SET HADR RESUME;
```

---

## AG Health Monitoring DMVs

### Replica and Database State
```sql
-- Overall AG health
SELECT
    ag.name AS ag_name,
    ar.replica_server_name,
    ar.availability_mode_desc,
    ar.failover_mode_desc,
    ars.role_desc AS current_role,
    ars.operational_state_desc,
    ars.connected_state_desc,
    ars.synchronization_health_desc,
    ars.last_connect_error_description
FROM sys.availability_groups ag
JOIN sys.availability_replicas ar ON ag.group_id = ar.group_id
JOIN sys.dm_hadr_availability_replica_states ars ON ar.replica_id = ars.replica_id
ORDER BY ag.name, ars.role_desc;

-- Database synchronization state
SELECT
    DB_NAME(drs.database_id) AS database_name,
    ar.replica_server_name,
    drs.is_local,
    drs.synchronization_state_desc,
    drs.synchronization_health_desc,
    drs.log_send_queue_size,       -- KB in send queue (should be near 0 for sync)
    drs.log_send_rate,             -- KB/sec being sent
    drs.redo_queue_size,           -- KB waiting to be redone on secondary
    drs.redo_rate,                 -- KB/sec being redone
    drs.last_commit_time,          -- Last commit time applied on this replica
    drs.last_redone_time,          -- Last log block fully redone
    drs.last_received_time         -- Last log block received
FROM sys.dm_hadr_database_replica_states drs
JOIN sys.availability_replicas ar ON drs.replica_id = ar.replica_id
ORDER BY drs.database_id, ar.replica_server_name;

-- Estimated data loss (async replicas)
SELECT
    DB_NAME(drs.database_id) AS database_name,
    ar.replica_server_name,
    ar.availability_mode_desc,
    drs.log_send_queue_size AS potential_data_loss_kb,
    drs.redo_queue_size AS lag_to_apply_kb,
    DATEDIFF(SECOND, drs.last_commit_time,
        (SELECT MAX(last_commit_time) FROM sys.dm_hadr_database_replica_states
         WHERE database_id = drs.database_id AND is_local = 0)) AS estimated_data_loss_sec
FROM sys.dm_hadr_database_replica_states drs
JOIN sys.availability_replicas ar ON drs.replica_id = ar.replica_id
WHERE ar.availability_mode = 0;  -- Async replicas only
```

### AG Listener and Connection Info
```sql
-- Listener info
SELECT
    ag.name AS ag_name,
    al.dns_name AS listener_name,
    ali.ip_address,
    ali.ip_subnet_mask,
    al.port
FROM sys.availability_group_listeners al
JOIN sys.availability_groups ag ON al.group_id = ag.group_id
JOIN sys.availability_group_listener_ip_addresses ali ON al.listener_id = ali.listener_id;
```

### AG Endpoint Status
```sql
-- Check mirroring endpoints (used by AG)
SELECT
    name,
    state_desc,
    role_desc,
    connection_auth_desc,
    encryption_algorithm_desc,
    port
FROM sys.database_mirroring_endpoints;
```

---

## Readable Secondary Configuration

```sql
-- Route connections to secondary for read-only (read scale-out)
-- Client app must use ApplicationIntent=ReadOnly in connection string
-- Example: Server=AG_Listener;Database=AppDB;ApplicationIntent=ReadOnly

-- Configure read-only routing
ALTER AVAILABILITY GROUP [AG_Production]
MODIFY REPLICA ON N'SQL01' WITH (
    PRIMARY_ROLE (READ_ONLY_ROUTING_LIST = ('SQL02', 'SQL03'))
);

ALTER AVAILABILITY GROUP [AG_Production]
MODIFY REPLICA ON N'SQL02' WITH (
    SECONDARY_ROLE (READ_ONLY_ROUTING_URL = N'TCP://SQL02.domain.com:1433'),
    SECONDARY_ROLE (ALLOW_CONNECTIONS = READ_ONLY)
);

ALTER AVAILABILITY GROUP [AG_Production]
MODIFY REPLICA ON N'SQL03' WITH (
    SECONDARY_ROLE (READ_ONLY_ROUTING_URL = N'TCP://SQL03.domain.com:1433'),
    SECONDARY_ROLE (ALLOW_CONNECTIONS = READ_ONLY)
);
```

---

## Backup Strategy for AG

```sql
-- Preferred backup target (run on all replicas; only executes on preferred)
-- Check if this replica is preferred for backups
IF sys.fn_hadr_backup_is_preferred_replica('AppDB1') = 1
BEGIN
    BACKUP DATABASE [AppDB1]
    TO DISK = N'\\Backup\AppDB1_Full.bak'
    WITH COMPRESSION, CHECKSUM, COPY_ONLY;  -- COPY_ONLY on secondary to avoid breaking log chain
END;

-- Log backup (must be on primary or capture from primary LSN chain)
-- In practice, run log backups on secondary with AUTOMATED_BACKUP_PREFERENCE = SECONDARY
BACKUP LOG [AppDB1] TO DISK = N'\\Backup\AppDB1_Log.trn' WITH COMPRESSION, CHECKSUM;
```

---

## Failover Cluster Instance (FCI)

### Architecture
- SQL Server runs on one active node at a time.
- Shared storage (SAN, S2D, Azure Shared Disk) holds data files — accessible to all nodes.
- Windows Server Failover Cluster manages node health; automatically restarts SQL on another node if active node fails.
- All databases (not a subset) fail over together.
- Storage becomes the single point of failure; use redundant storage.

### FCI Health Check
```sql
-- Check FCI cluster configuration
SELECT * FROM sys.dm_os_cluster_nodes;
SELECT * FROM sys.dm_os_cluster_properties;

-- Active node
SELECT SERVERPROPERTY('ComputerNamePhysicalNetBIOS') AS active_node;
```

---

## Log Shipping

### Architecture
- Automated backup, copy, and restore of transaction log backups.
- Primary sends log backups to a share; monitor server (optional) watches for failures.
- Secondary databases remain in STANDBY (readable) or NORECOVERY (not readable).
- Multiple secondary servers supported.

### Configure Log Shipping (T-SQL)
```sql
-- On primary server:
-- Initialize with a full backup
BACKUP DATABASE [PrimaryDB] TO DISK = '\\Share\PrimaryDB_Full.bak' WITH INIT;

-- Add primary database to log shipping
EXEC master.dbo.sp_add_log_shipping_primary_database
    @database = N'PrimaryDB',
    @backup_directory = N'D:\LogBackups',
    @backup_share = N'\\SQL01\LogBackups',
    @backup_job_name = N'LSBackup_PrimaryDB',
    @backup_retention_period = 4320,  -- 3 days in minutes
    @backup_threshold = 60,           -- Alert if no backup in 60 minutes
    @threshold_alert_enabled = 1,
    @history_retention_period = 5760; -- 4 days in minutes

-- On secondary server:
-- Restore the full backup first
RESTORE DATABASE [SecondaryDB]
FROM DISK = '\\Share\PrimaryDB_Full.bak'
WITH NORECOVERY, MOVE 'PrimaryDB' TO 'E:\Data\SecondaryDB.mdf',
     MOVE 'PrimaryDB_log' TO 'E:\Log\SecondaryDB_log.ldf';

-- Add secondary database
EXEC master.dbo.sp_add_log_shipping_secondary_primary
    @primary_server = N'SQL01',
    @primary_database = N'PrimaryDB',
    @backup_source_directory = N'\\SQL01\LogBackups',
    @backup_destination_directory = N'D:\LogCopy',
    @copy_job_name = N'LSCopy_SQL01_PrimaryDB',
    @restore_job_name = N'LSRestore_SQL01_PrimaryDB';

EXEC master.dbo.sp_add_log_shipping_secondary_database
    @secondary_database = N'SecondaryDB',
    @primary_server = N'SQL01',
    @primary_database = N'PrimaryDB',
    @restore_delay = 0,              -- Minutes to delay before restoring (poison pill delay)
    @restore_mode = 0,               -- 0 = NORECOVERY (not readable), 1 = STANDBY (readable)
    @disconnect_users = 0,
    @restore_threshold = 90,         -- Alert if no restore in 90 minutes
    @threshold_alert_enabled = 1,
    @history_retention_period = 5760;
```

### Log Shipping Health Check
```sql
-- On primary
SELECT * FROM msdb.dbo.log_shipping_primary_databases;
SELECT * FROM msdb.dbo.log_shipping_monitor_primary;

-- On secondary
SELECT * FROM msdb.dbo.log_shipping_secondary_databases;
SELECT * FROM msdb.dbo.log_shipping_monitor_secondary;

-- History (check for failures)
SELECT * FROM msdb.dbo.log_shipping_monitor_history_detail
WHERE error_number > 0
ORDER BY log_time DESC;
```

---

## Database Mirroring (Deprecated — for legacy awareness)

```sql
-- Deprecated since SQL Server 2012; removed from future versions
-- Use Always On AGs instead. Documentation for maintaining existing setups only.

-- Check mirroring state
SELECT
    db_name(database_id) AS database_name,
    mirroring_state_desc,
    mirroring_role_desc,
    mirroring_safety_level_desc,  -- FULL=sync, OFF=async
    mirroring_partner_name,
    mirroring_witness_name,
    mirroring_witness_state_desc
FROM sys.database_mirroring
WHERE mirroring_guid IS NOT NULL;
```

---

## Stretch Database (Legacy — deprecated in SQL 2022)

Stretch Database was removed in SQL Server 2022. Use Azure Synapse Link or export/archive patterns instead.

---

## Monitoring and Alerting for HA/DR

### Critical Alerts to Configure

```sql
-- 1. AG role change (failover occurred)
-- Alert on SQL Server Alert: "Availability group role change"
-- OR WMI event / Extended Event: hadr_ag_manager

-- 2. Secondary replica not synchronizing
SELECT
    DB_NAME(drs.database_id) AS db_name,
    ar.replica_server_name,
    drs.synchronization_state_desc,
    drs.redo_queue_size
FROM sys.dm_hadr_database_replica_states drs
JOIN sys.availability_replicas ar ON drs.replica_id = ar.replica_id
WHERE drs.synchronization_state_desc NOT IN ('SYNCHRONIZED', 'SYNCHRONIZING')
  AND ar.replica_server_name != @@SERVERNAME;

-- 3. AG listener connections (check if listener is responding)
SELECT
    al.dns_name,
    ags.primary_replica,
    ags.synchronization_health_desc
FROM sys.availability_group_listeners al
JOIN sys.dm_hadr_availability_group_states ags ON al.group_id = ags.group_id;

-- 4. Log shipping lag alert
SELECT
    primary_database,
    secondary_database,
    restore_lag,      -- Minutes behind
    restore_threshold
FROM msdb.dbo.log_shipping_monitor_secondary
WHERE restore_lag > restore_threshold;

-- 5. FCI node failures
SELECT * FROM sys.dm_os_cluster_nodes WHERE is_current_owner = 1;
```

### WSFC Quorum Check (PowerShell — relevant context)
```powershell
# Check WSFC quorum mode and status
Get-ClusterQuorum
Get-ClusterNode | Select-Object Name, State

# Add cloud witness (Azure Storage) for two-node clusters
Set-ClusterQuorum -CloudWitness -AccountName "storageaccount" -AccessKey "key=="
```

---

## Read Scale-Out (SQL 2016+ — No Cluster)

```sql
-- Basic AG without WSFC for read scale-out (no automatic failover)
CREATE AVAILABILITY GROUP [AG_ReadScale]
WITH (CLUSTER_TYPE = NONE, DB_FAILOVER = OFF)  -- No WSFC
FOR DATABASE [ReportDB]
REPLICA ON
    N'SQL01' WITH (
        ENDPOINT_URL = N'TCP://SQL01:5022',
        AVAILABILITY_MODE = ASYNCHRONOUS_COMMIT,
        FAILOVER_MODE = MANUAL,
        SEEDING_MODE = AUTOMATIC,
        SECONDARY_ROLE (ALLOW_CONNECTIONS = READ_ONLY)
    ),
    N'SQL02' WITH (
        ENDPOINT_URL = N'TCP://SQL02:5022',
        AVAILABILITY_MODE = ASYNCHRONOUS_COMMIT,
        FAILOVER_MODE = MANUAL,
        SEEDING_MODE = AUTOMATIC,
        SECONDARY_ROLE (ALLOW_CONNECTIONS = READ_ONLY)
    );
```

---

## Distributed Availability Groups (DAG) — SQL 2016+

```sql
-- AG spanning two independent WSFC clusters (geo-DR)
CREATE AVAILABILITY GROUP [DAG_Production]
WITH (DISTRIBUTED)
AVAILABILITY GROUP ON
    N'AG_Primary' WITH (
        LISTENER_URL = N'TCP://AG_Listener.domain.com:5022',
        AVAILABILITY_MODE = ASYNCHRONOUS_COMMIT,
        FAILOVER_MODE = MANUAL,
        SEEDING_MODE = AUTOMATIC
    ),
    N'AG_DR' WITH (
        LISTENER_URL = N'TCP://AG_DR_Listener.dr.domain.com:5022',
        AVAILABILITY_MODE = ASYNCHRONOUS_COMMIT,
        FAILOVER_MODE = MANUAL,
        SEEDING_MODE = AUTOMATIC
    );
```

---

## RTO / RPO Design Guidance

### Tier 1 — Mission Critical (RPO 0, RTO < 30 sec)
- Synchronous AG with automatic failover
- Multiple sync replicas for local HA
- Async replica for DR site
- Persistent client retry logic to handle brief connection interruption during failover

### Tier 2 — Business Critical (RPO < 15 min, RTO < 30 min)
- Asynchronous AG (async within same datacenter for scale-out)
- Log shipping to DR site with 15-minute backup interval
- Manual failover procedure documented and tested quarterly

### Tier 3 — Standard (RPO < 24 hours, RTO < 4 hours)
- Nightly full backup + log backups every hour
- Offsite backup copy (Azure Blob, tape, secondary datacenter)
- Documented restore procedure, tested semi-annually

---

## Best Practices

1. **Test failover regularly**: Perform planned failovers quarterly; confirm RTO meets SLA.
2. **Test restores**: Verify backups monthly by restoring to a test environment.
3. **Use synchronous mode for RPO=0**: Async mode accepts data loss risk.
4. **Quorum configuration**: Use cloud witness or file share witness for 2-node clusters; never rely on node majority alone with even node counts.
5. **Network redundancy**: Use multiple NIC teaming for AG/FCI traffic.
6. **Dedicated endpoint port**: Use TCP port 5022 for AG endpoints; keep separate from SQL data port 1433.
7. **Monitor redo queue**: High redo queue on secondary indicates secondary is falling behind.
8. **Listener configuration**: Use `MultiSubnetFailover=True` in connection strings for multi-subnet AGs.
9. **Backup on secondary**: Configure `AUTOMATED_BACKUP_PREFERENCE = SECONDARY` to reduce primary load.
10. **Document procedures**: Maintain runbooks for all failover and failback scenarios.
11. **Service Broker and MSDTC**: Coordinate these with AG team; they require special handling.
12. **DTC and AGs**: Distributed transactions require DTC configuration; test thoroughly.
