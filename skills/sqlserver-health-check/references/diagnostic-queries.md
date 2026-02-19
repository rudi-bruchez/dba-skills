# SQL Server Health Check — Diagnostic Queries

Complete T-SQL scripts for all health check areas. Run these in SSMS or Azure Data Studio against the target instance.

---

## Instance Information

```sql
-- SQL Server version, edition, uptime
SELECT
    SERVERPROPERTY('ProductVersion')                          AS sql_version,
    SERVERPROPERTY('ProductLevel')                           AS product_level,
    SERVERPROPERTY('Edition')                                AS edition,
    SERVERPROPERTY('InstanceName')                           AS instance_name,
    SERVERPROPERTY('ComputerNamePhysicalNetBIOS')            AS physical_node,
    (SELECT ms_ticks / 1000 / 60 / 60 FROM sys.dm_os_sys_info) AS uptime_hours,
    (SELECT cpu_count FROM sys.dm_os_sys_info)               AS logical_cpu_count,
    (SELECT physical_memory_kb / 1024 FROM sys.dm_os_sys_info) AS physical_memory_mb;

-- SQL Server configuration
SELECT name, value_in_use, description
FROM sys.configurations
WHERE name IN (
    'max server memory (MB)', 'min server memory (MB)',
    'max degree of parallelism', 'cost threshold for parallelism',
    'optimize for ad hoc workloads', 'backup compression default',
    'remote admin connections', 'cross db ownership chaining',
    'clr enabled', 'Ole Automation Procedures', 'xp_cmdshell'
)
ORDER BY name;
```

---

## Wait Statistics

```sql
-- Top 20 wait types (benign idle waits excluded)
SELECT TOP 20
    wait_type,
    wait_time_ms / 1000.0                              AS wait_time_sec,
    (wait_time_ms - signal_wait_time_ms) / 1000.0     AS resource_wait_sec,
    signal_wait_time_ms / 1000.0                       AS signal_wait_sec,
    waiting_tasks_count,
    CASE WHEN waiting_tasks_count = 0 THEN 0
         ELSE wait_time_ms / waiting_tasks_count END   AS avg_wait_ms,
    CAST(100.0 * wait_time_ms / SUM(wait_time_ms) OVER ()
         AS DECIMAL(5,2))                              AS pct_of_total
FROM sys.dm_os_wait_stats
WHERE wait_type NOT IN (
    'SLEEP_TASK','BROKER_TO_FLUSH','BROKER_TASK_STOP','CLR_AUTO_EVENT',
    'DISPATCHER_QUEUE_SEMAPHORE','FT_IFTS_SCHEDULER_IDLE_WAIT',
    'HADR_FILESTREAM_IOMGR_IOCOMPLETION','HADR_WORK_QUEUE',
    'LAZYWRITER_SLEEP','LOGMGR_QUEUE','ONDEMAND_TASK_QUEUE',
    'REQUEST_FOR_DEADLOCK_SEARCH','RESOURCE_QUEUE','SERVER_IDLE_CHECK',
    'SLEEP_DBSTARTUP','SLEEP_DCOMSTARTUP','SLEEP_MASTERDBREADY',
    'SLEEP_MASTERMDREADY','SLEEP_MASTERUPGRADED','SLEEP_MSDBSTARTUP',
    'SLEEP_SYSTEMTASK','SLEEP_TEMPDBSTARTUP','SNI_HTTP_ACCEPT',
    'SP_SERVER_DIAGNOSTICS_SLEEP','SQLTRACE_BUFFER_FLUSH',
    'SQLTRACE_INCREMENTAL_FLUSH_SLEEP','WAITFOR','XE_DISPATCHER_WAIT',
    'XE_TIMER_EVENT','BROKER_EVENTHANDLER','CHECKPOINT_QUEUE',
    'DBMIRROR_EVENTS_QUEUE','SQLTRACE_WAIT_ENTRIES',
    'WAIT_XTP_OFFLINE_CKPT_NEW_LOG'
)
ORDER BY wait_time_ms DESC;

-- Reset wait stats (use cautiously on production — clears historical baseline)
-- DBCC SQLPERF('sys.dm_os_wait_stats', CLEAR);

-- Signal wait percentage (CPU pressure indicator)
SELECT
    SUM(signal_wait_time_ms) * 100.0 / NULLIF(SUM(wait_time_ms), 0)
        AS signal_wait_pct
FROM sys.dm_os_wait_stats
WHERE wait_type NOT LIKE 'SLEEP_%'
  AND wait_type NOT IN ('WAITFOR', 'XE_TIMER_EVENT', 'XE_DISPATCHER_WAIT');
-- > 25%: potential CPU pressure
```

---

## Blocking and Locking

```sql
-- Currently blocked sessions
SELECT
    r.session_id,
    r.blocking_session_id,
    r.wait_type,
    r.wait_time / 1000.0   AS wait_sec,
    r.status,
    DB_NAME(r.database_id) AS database_name,
    SUBSTRING(st.text,
        (r.statement_start_offset / 2) + 1,
        ((CASE r.statement_end_offset WHEN -1 THEN DATALENGTH(st.text)
          ELSE r.statement_end_offset END - r.statement_start_offset) / 2) + 1)
        AS current_statement,
    s.login_name,
    s.host_name,
    s.program_name,
    s.open_transaction_count
FROM sys.dm_exec_requests r
JOIN sys.dm_exec_sessions s ON r.session_id = s.session_id
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) st
WHERE r.blocking_session_id > 0
ORDER BY r.wait_time DESC;

-- Head blockers (blocking others but not themselves blocked)
SELECT r.blocking_session_id AS head_blocker,
       s.login_name, s.host_name, s.program_name,
       s.open_transaction_count,
       SUBSTRING(st.text, 1, 500) AS head_blocker_sql
FROM sys.dm_exec_sessions s
LEFT JOIN sys.dm_exec_requests r ON s.session_id = r.session_id
CROSS APPLY sys.dm_exec_sql_text(COALESCE(r.sql_handle, s.sql_handle)) st
WHERE s.session_id IN (
    SELECT DISTINCT blocking_session_id FROM sys.dm_exec_requests WHERE blocking_session_id > 0
)
  AND s.session_id NOT IN (
    SELECT session_id FROM sys.dm_exec_requests WHERE blocking_session_id > 0
);

-- Lock details (current lock holders and waiters)
SELECT
    tl.resource_type,
    tl.resource_database_id,
    tl.resource_associated_entity_id,
    tl.resource_description,
    tl.request_mode,
    tl.request_type,
    tl.request_status,
    tl.request_session_id,
    s.login_name,
    s.host_name
FROM sys.dm_tran_locks tl
JOIN sys.dm_exec_sessions s ON tl.request_session_id = s.session_id
WHERE tl.resource_type <> 'DATABASE'
ORDER BY tl.resource_type, tl.request_status;

-- Deadlocks from system_health extended event session
SELECT
    xdr.value('@timestamp', 'datetime2') AS deadlock_time,
    xdr.query('.')                        AS deadlock_graph
FROM (
    SELECT CAST(target_data AS XML) AS target_data
    FROM sys.dm_xe_session_targets t
    JOIN sys.dm_xe_sessions s ON t.event_session_address = s.address
    WHERE s.name = 'system_health' AND t.target_name = 'ring_buffer'
) AS data
CROSS APPLY target_data.nodes('//RingBufferTarget/event[@name="xml_deadlock_report"]') AS xdt(xdr)
ORDER BY deadlock_time DESC;
```

---

## Memory

```sql
-- Page Life Expectancy
SELECT object_name, counter_name, cntr_value AS page_life_expectancy
FROM sys.dm_os_performance_counters
WHERE counter_name = 'Page life expectancy'
  AND object_name LIKE '%Buffer Manager%';

-- Memory breakdown by clerk (top 20)
SELECT TOP 20
    type AS clerk_type,
    SUM(pages_kb) / 1024.0 AS memory_mb
FROM sys.dm_os_memory_clerks
GROUP BY type
ORDER BY SUM(pages_kb) DESC;

-- Buffer pool usage by database
SELECT
    DB_NAME(database_id) AS database_name,
    COUNT(*) * 8 / 1024.0 AS buffer_pool_mb
FROM sys.dm_os_buffer_descriptors
WHERE database_id > 0
GROUP BY database_id
ORDER BY buffer_pool_mb DESC;

-- Process memory
SELECT
    physical_memory_in_use_kb / 1024.0     AS physical_memory_in_use_mb,
    locked_page_allocations_kb / 1024.0    AS locked_pages_mb,
    virtual_address_space_committed_kb / 1024.0 AS virtual_committed_mb,
    process_physical_memory_low,
    process_virtual_memory_low
FROM sys.dm_os_process_memory;

-- Single-use plan cache (ad hoc SQL pollution)
SELECT
    COUNT(*) AS single_use_plan_count,
    SUM(size_in_bytes) / 1024.0 / 1024.0 AS total_size_mb
FROM sys.dm_exec_cached_plans
WHERE usecounts = 1 AND objtype = 'Adhoc';
```

---

## CPU

```sql
-- CPU usage history from ring buffer (last 30 samples, ~30 minutes)
SELECT TOP 30
    record.value('(./Record/SchedulerMonitorEvent/SystemHealth/ProcessUtilization)[1]', 'int')
        AS sql_cpu_pct,
    record.value('(./Record/SchedulerMonitorEvent/SystemHealth/SystemIdle)[1]', 'int')
        AS system_idle_pct,
    100 - record.value('(./Record/SchedulerMonitorEvent/SystemHealth/SystemIdle)[1]', 'int')
        - record.value('(./Record/SchedulerMonitorEvent/SystemHealth/ProcessUtilization)[1]', 'int')
        AS other_cpu_pct,
    DATEADD(ms, -1 * (ts_now - [timestamp]), GETDATE()) AS event_time
FROM (
    SELECT [timestamp], CONVERT(xml, record) AS record, ts_now
    FROM sys.dm_os_ring_buffers
    CROSS JOIN (SELECT ms_ticks AS ts_now FROM sys.dm_os_sys_info) t
    WHERE ring_buffer_type = N'RING_BUFFER_SCHEDULER_MONITOR'
      AND record LIKE '%<SystemHealth>%'
) AS ring_data
ORDER BY event_time DESC;

-- Scheduler runnable queue (CPU pressure at scheduler level)
SELECT scheduler_id, current_tasks_count, runnable_tasks_count,
       work_queue_count, pending_disk_io_count, active_workers_count
FROM sys.dm_os_schedulers
WHERE status = 'VISIBLE ONLINE'
ORDER BY runnable_tasks_count DESC;
-- runnable_tasks_count > 2 on any scheduler = CPU pressure
```

---

## Disk I/O

```sql
-- I/O latency per database file (cumulative since last restart)
SELECT
    DB_NAME(fs.database_id)                                         AS database_name,
    mf.physical_name,
    mf.type_desc                                                    AS file_type,
    fs.io_stall_read_ms  / NULLIF(fs.num_of_reads, 0)              AS avg_read_ms,
    fs.io_stall_write_ms / NULLIF(fs.num_of_writes, 0)             AS avg_write_ms,
    fs.io_stall          / NULLIF(fs.num_of_reads + fs.num_of_writes, 0) AS avg_io_ms,
    fs.num_of_reads,
    fs.num_of_writes,
    fs.io_stall_read_ms,
    fs.io_stall_write_ms,
    fs.io_stall
FROM sys.dm_io_virtual_file_stats(NULL, NULL) fs
JOIN sys.master_files mf
    ON fs.database_id = mf.database_id AND fs.file_id = mf.file_id
ORDER BY avg_io_ms DESC;

-- Disk volume free space
SELECT DISTINCT
    volume_mount_point,
    total_bytes / 1024.0 / 1024 / 1024    AS total_gb,
    available_bytes / 1024.0 / 1024 / 1024 AS free_gb,
    CAST(available_bytes * 100.0 / total_bytes AS DECIMAL(5,2)) AS free_pct
FROM sys.dm_os_volume_stats(1, 1)
CROSS JOIN sys.master_files mf
CROSS APPLY sys.dm_os_volume_stats(mf.database_id, mf.file_id)
ORDER BY free_pct ASC;
```

---

## Database State and Configuration

```sql
-- Database status, recovery model, options
SELECT
    d.name,
    d.state_desc,
    d.recovery_model_desc,
    d.compatibility_level,
    d.is_auto_shrink_on,
    d.is_auto_close_on,
    d.is_auto_create_stats_on,
    d.is_auto_update_stats_on,
    d.page_verify_option_desc,
    d.log_reuse_wait_desc,
    d.create_date,
    d.collation_name
FROM sys.databases d
ORDER BY d.name;

-- File sizes, growth settings
SELECT
    DB_NAME(database_id)   AS database_name,
    name                   AS logical_name,
    physical_name,
    type_desc,
    CAST(size * 8.0 / 1024 AS DECIMAL(10,2)) AS size_mb,
    CASE max_size
        WHEN -1 THEN 'Unlimited'
        WHEN 0  THEN 'No growth'
        ELSE CAST(max_size * 8.0 / 1024 AS VARCHAR(20)) + ' MB'
    END AS max_size,
    CASE is_percent_growth
        WHEN 1 THEN CAST(growth AS VARCHAR(10)) + '%'
        ELSE CAST(growth * 8.0 / 1024 AS VARCHAR(20)) + ' MB'
    END AS growth_setting
FROM sys.master_files
ORDER BY database_id, file_id;

-- VLF count per database (SQL 2016+)
SELECT
    DB_NAME(database_id) AS database_name,
    COUNT(*)             AS vlf_count
FROM sys.dm_db_log_info(NULL)
GROUP BY database_id
ORDER BY vlf_count DESC;
-- > 1000 VLFs: log fragmentation problem

-- Log space usage
SELECT
    DB_NAME()       AS database_name,
    total_log_size_mb,
    used_log_space_mb,
    used_log_space_percent,
    log_backup_time AS last_log_backup
FROM sys.dm_db_log_space_usage;
```

---

## Extended Events — Long-Running Query Capture

```sql
-- Create a session to capture queries running > 5 seconds
CREATE EVENT SESSION [LongRunningQueries] ON SERVER
ADD EVENT sqlserver.sql_statement_completed (
    WHERE duration > 5000000  -- 5 seconds in microseconds
    ACTION(sqlserver.sql_text, sqlserver.database_name,
           sqlserver.username, sqlserver.plan_handle)
),
ADD EVENT sqlserver.rpc_completed (
    WHERE duration > 5000000
    ACTION(sqlserver.sql_text, sqlserver.database_name, sqlserver.username)
)
ADD TARGET package0.ring_buffer (SET max_memory = 51200)
WITH (MAX_DISPATCH_LATENCY = 30 SECONDS);

ALTER EVENT SESSION [LongRunningQueries] ON SERVER STATE = START;

-- Read from the ring buffer
SELECT
    xdr.value('@name', 'varchar(50)')        AS event_name,
    xdr.value('@timestamp', 'datetime2')     AS event_time,
    xdr.value('(data[@name="duration"]/value)[1]', 'bigint') / 1000000.0 AS duration_sec,
    xdr.value('(action[@name="sql_text"]/value)[1]', 'nvarchar(max)')  AS sql_text,
    xdr.value('(action[@name="database_name"]/value)[1]', 'nvarchar(128)') AS database_name
FROM (
    SELECT CAST(target_data AS XML) AS target_data
    FROM sys.dm_xe_session_targets t
    JOIN sys.dm_xe_sessions s ON t.event_session_address = s.address
    WHERE s.name = 'LongRunningQueries'
) AS data
CROSS APPLY target_data.nodes('//RingBufferTarget/event') AS xdt(xdr)
ORDER BY event_time DESC;
```
