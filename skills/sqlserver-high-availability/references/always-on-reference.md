# Always On Availability Groups — Complete Reference

## Prerequisites

```sql
-- Verify HADR is enabled on the SQL Server instance
SELECT SERVERPROPERTY('IsHadrEnabled') AS hadr_enabled;
-- Must return 1. If 0, enable via SQL Server Configuration Manager or PowerShell:
-- Enable-SqlAlwaysOn -ServerInstance "SQL01" -Force  (requires service restart)

-- Verify mirroring endpoint exists
SELECT name, state_desc, role_desc, port, encryption_algorithm_desc
FROM sys.database_mirroring_endpoints;
-- If missing, create it:
CREATE ENDPOINT [Hadr_endpoint]
    STATE = STARTED
    AS TCP (LISTENER_PORT = 5022)
    FOR DATABASE_MIRRORING (ROLE = ALL, AUTHENTICATION = WINDOWS NEGOTIATE, ENCRYPTION = REQUIRED ALGORITHM AES);
```

---

## Create an Availability Group

```sql
-- Run on the intended primary replica
CREATE AVAILABILITY GROUP [AG_Production]
WITH (
    AUTOMATED_BACKUP_PREFERENCE = SECONDARY,  -- Prefer backups on secondary
    FAILURE_CONDITION_LEVEL = 3,              -- 3 = default: server down or unresponsive
    HEALTH_CHECK_TIMEOUT = 30000,             -- ms before triggering auto-failover health check
    DB_FAILOVER = ON,                         -- SQL 2016+: database-level health triggers AG failover
    DTC_SUPPORT = NONE                        -- Set to PER_DB for distributed transactions (SQL 2016+)
)
FOR DATABASE [AppDB1], [AppDB2]
REPLICA ON
    N'SQL01' WITH (
        ENDPOINT_URL = N'TCP://SQL01.domain.com:5022',
        AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,   -- RPO = 0
        FAILOVER_MODE = AUTOMATIC,                -- Auto-failover with WSFC
        SEEDING_MODE = AUTOMATIC,                 -- SQL 2016+: auto-seed secondary
        SECONDARY_ROLE (ALLOW_CONNECTIONS = NO)
    ),
    N'SQL02' WITH (
        ENDPOINT_URL = N'TCP://SQL02.domain.com:5022',
        AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
        FAILOVER_MODE = AUTOMATIC,
        SEEDING_MODE = AUTOMATIC,
        SECONDARY_ROLE (ALLOW_CONNECTIONS = READ_ONLY)  -- Readable secondary
    ),
    N'SQL03' WITH (  -- DR site — async replica
        ENDPOINT_URL = N'TCP://SQL03.domain.com:5022',
        AVAILABILITY_MODE = ASYNCHRONOUS_COMMIT,  -- Potential data loss on failover
        FAILOVER_MODE = MANUAL,
        SEEDING_MODE = AUTOMATIC,
        SECONDARY_ROLE (ALLOW_CONNECTIONS = READ_ONLY)
    );

-- Add a listener (virtual network name clients connect to)
ALTER AVAILABILITY GROUP [AG_Production]
ADD LISTENER N'AG_Listener' (
    WITH IP ((N'10.0.1.50', N'255.255.255.0')),
    PORT = 1433
);
```

---

## Join Secondary Replicas

```sql
-- Run on each secondary replica after the AG is created on primary
ALTER AVAILABILITY GROUP [AG_Production] JOIN;
ALTER AVAILABILITY GROUP [AG_Production] GRANT CREATE ANY DATABASE;
-- With SEEDING_MODE = AUTOMATIC, databases are seeded automatically.
-- Monitor seeding progress:
SELECT start_time, completion_time, is_master, current_state, failure_message,
       database_name, totalMBytes, transferredMBytes
FROM sys.dm_hadr_automatic_seeding;
```

---

## Modify Replica Configuration

```sql
-- Change a sync replica to async (e.g., for a DR site)
ALTER AVAILABILITY GROUP [AG_Production]
MODIFY REPLICA ON N'SQL03' WITH (
    AVAILABILITY_MODE = ASYNCHRONOUS_COMMIT,
    FAILOVER_MODE = MANUAL
);

-- Allow or restrict connections on secondary
ALTER AVAILABILITY GROUP [AG_Production]
MODIFY REPLICA ON N'SQL02' WITH (
    SECONDARY_ROLE (ALLOW_CONNECTIONS = READ_ONLY)
);

-- Change backup preference
ALTER AVAILABILITY GROUP [AG_Production]
WITH (AUTOMATED_BACKUP_PREFERENCE = SECONDARY);
```

---

## Add or Remove Databases

```sql
-- Add a new database to an existing AG (database must be in FULL recovery model
-- and backed up before running this)
ALTER AVAILABILITY GROUP [AG_Production] ADD DATABASE [NewDB];

-- Remove a database from the AG
ALTER AVAILABILITY GROUP [AG_Production] REMOVE DATABASE [OldDB];
-- After removal, the database on secondaries enters RESTORING state.
-- To bring it online as standalone: RESTORE DATABASE [OldDB] WITH RECOVERY;
```

---

## Readable Secondary Configuration

```sql
-- Configure read-only routing so that connections with ApplicationIntent=ReadOnly
-- are redirected from the listener to a readable secondary.

-- On primary: set routing list (ordered preference)
ALTER AVAILABILITY GROUP [AG_Production]
MODIFY REPLICA ON N'SQL01' WITH (
    PRIMARY_ROLE (READ_ONLY_ROUTING_LIST = ('SQL02', 'SQL03'))
);

-- On each readable secondary: set the routing URL
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

-- Client connection string for read-only routing:
-- Server=AG_Listener,1433;Database=AppDB1;ApplicationIntent=ReadOnly;MultiSubnetFailover=True
```

---

## Read Scale-Out AG (No Cluster — SQL 2016+)

```sql
-- AG without WSFC for read scale-out only (no automatic failover)
CREATE AVAILABILITY GROUP [AG_ReadScale]
WITH (CLUSTER_TYPE = NONE, DB_FAILOVER = OFF)
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

## Distributed Availability Groups (DAG — SQL 2016+)

Span two independent WSFC clusters for geo-DR scenarios.

```sql
-- Step 1: Create the DAG on the primary AG's primary replica
CREATE AVAILABILITY GROUP [DAG_Production]
WITH (DISTRIBUTED)
AVAILABILITY GROUP ON
    N'AG_Production' WITH (
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

-- Step 2: Join the DAG on the DR site's primary replica
ALTER AVAILABILITY GROUP [DAG_Production]
JOIN AVAILABILITY GROUP ON
    N'AG_Production' WITH (
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

## Backup Strategy for AGs

```sql
-- Run on all replicas (SQL Agent job); only executes backup on the preferred replica
IF sys.fn_hadr_backup_is_preferred_replica('AppDB1') = 1
BEGIN
    -- Full backup on secondary: use COPY_ONLY to preserve the differential base on primary
    BACKUP DATABASE [AppDB1]
    TO DISK = N'\\Backup\AppDB1_Full.bak'
    WITH COMPRESSION, CHECKSUM, COPY_ONLY;
END;

-- Transaction log backups: run on preferred replica (breaks log chain on that replica)
IF sys.fn_hadr_backup_is_preferred_replica('AppDB1') = 1
BEGIN
    BACKUP LOG [AppDB1]
    TO DISK = N'\\Backup\AppDB1_Log.trn'
    WITH COMPRESSION, CHECKSUM;
END;
```

---

## WSFC Quorum Management (PowerShell)

```powershell
# Check current quorum mode and status
Get-ClusterQuorum
Get-ClusterNode | Select-Object Name, State, NodeWeight

# Recommended: cloud witness for two-node clusters (no split-brain)
Set-ClusterQuorum -CloudWitness -AccountName "storageaccount" -AccessKey "base64key=="

# Check cluster network
Get-ClusterNetwork | Select-Object Name, State, Role

# View cluster events
Get-ClusterLog -Destination C:\Temp\ClusterLog -TimeSpan 60  # Last 60 minutes
```

---

## Key DMVs for Always On AGs

| DMV | Purpose |
|---|---|
| `sys.availability_groups` | AG names, settings, backup preference |
| `sys.availability_replicas` | Replica configuration (mode, failover mode, endpoint) |
| `sys.dm_hadr_availability_replica_states` | Runtime replica health (role, sync health, connected state) |
| `sys.dm_hadr_database_replica_states` | Per-database sync state, queue sizes, rates |
| `sys.dm_hadr_availability_group_states` | AG-level health and primary replica name |
| `sys.availability_group_listeners` | Listener names and port |
| `sys.availability_group_listener_ip_addresses` | Listener IP addresses |
| `sys.database_mirroring_endpoints` | Endpoint state and port |
| `sys.dm_hadr_automatic_seeding` | Automatic seeding progress |
| `sys.dm_os_cluster_nodes` | WSFC node status (for FCI) |
