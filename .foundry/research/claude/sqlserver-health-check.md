# SQL Server Health Checks & Diagnostics

## Overview

SQL Server exposes health data through Dynamic Management Views (DMVs), system catalog views, and system stored procedures. Health checks cover CPU, memory, I/O, wait statistics, blocking, and configuration. This document covers the key DMVs, diagnostic queries, thresholds, and common issues.

---

## Key System Views and DMVs

### Wait Statistics
- `sys.dm_os_wait_stats` — Cumulative wait statistics since last restart or reset. Core of health analysis.
- `sys.dm_exec_session_wait_stats` — Wait stats per active session (SQL 2016+).
- `sys.dm_os_waiting_tasks` — Currently waiting tasks with wait type and blocking session.

### CPU and Scheduler
- `sys.dm_os_schedulers` — CPU schedulers: runnable queue length, pending I/O, yield count.
- `sys.dm_os_ring_buffers` — Ring buffer data including CPU usage signals (type = 'RING_BUFFER_SCHEDULER_MONITOR').
- `sys.dm_exec_requests` — Active requests with CPU time, reads, writes, wait type, blocking session.
- `sys.dm_os_sys_info` — CPU count, memory totals, server uptime.

### Memory
- `sys.dm_os_memory_clerks` — Memory allocated per component (buffer pool, plan cache, CLR, etc.).
- `sys.dm_os_buffer_descriptors` — Individual buffer pool pages (expensive to query).
- `sys.dm_os_process_memory` — SQL process virtual/physical memory usage.
- `sys.dm_os_performance_counters` — Windows performance counters accessible via T-SQL.
- `sys.dm_os_memory_brokers` — Memory allocation decisions by broker.

### Sessions and Connections
- `sys.dm_exec_sessions` — All sessions: login, host, database, CPU, memory, status.
- `sys.dm_exec_connections` — Network connection info per session.
- `sys.dm_exec_requests` — Currently executing requests.
- `sys.dm_exec_sql_text(sql_handle)` — Retrieves SQL text from handle.
- `sys.dm_exec_query_plan(plan_handle)` — Retrieves XML execution plan.

### Blocking and Deadlocks
- `sys.dm_os_waiting_tasks` — Real-time blocking chains.
- `sys.dm_tran_locks` — Current lock holders and waiters.
- `sys.dm_exec_requests` — blocking_session_id column identifies head blocker.
- System Health Extended Event session — captures deadlock graphs automatically.

### Disk I/O
- `sys.dm_io_virtual_file_stats(db_id, file_id)` — I/O latency per database file.
- `sys.dm_os_volume_stats(db_id, file_id)` — Disk volume free space.
- `sys.dm_io_pending_io_requests` — Pending I/O requests.

### Database State
- `sys.databases` — Database status, recovery model, compatibility level, mirroring state.
- `sys.dm_db_mirroring_auto_page_repair` — Page repair events.
- `sys.dm_hadr_availability_replica_states` — Always On replica health.
- `sys.dm_db_index_physical_stats` — Index fragmentation.

---

## Core Diagnostic Queries

### 1. Top Wait Statistics (baseline health)
```sql
-- Top 20 wait types, excluding benign waits
SELECT TOP 20
    wait_type,
    wait_time_ms / 1000.0 AS wait_time_sec,
    (wait_time_ms - signal_wait_time_ms) / 1000.0 AS resource_wait_sec,
    signal_wait_time_ms / 1000.0 AS signal_wait_sec,
    waiting_tasks_count,
    CASE WHEN waiting_tasks_count = 0 THEN 0
         ELSE wait_time_ms / waiting_tasks_count
    END AS avg_wait_ms
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
    'DBMIRROR_EVENTS_QUEUE','SQLTRACE_WAIT_ENTRIES','WAIT_XTP_OFFLINE_CKPT_NEW_LOG'
)
ORDER BY wait_time_ms DESC;
```

### 2. Current Blocking Chains
```sql
SELECT
    r.session_id,
    r.blocking_session_id,
    r.wait_type,
    r.wait_time / 1000.0 AS wait_sec,
    r.status,
    DB_NAME(r.database_id) AS database_name,
    SUBSTRING(st.text, (r.statement_start_offset/2)+1,
        ((CASE r.statement_end_offset WHEN -1 THEN DATALENGTH(st.text)
          ELSE r.statement_end_offset END - r.statement_start_offset)/2)+1) AS current_statement,
    s.login_name,
    s.host_name,
    s.program_name
FROM sys.dm_exec_requests r
JOIN sys.dm_exec_sessions s ON r.session_id = s.session_id
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) st
WHERE r.blocking_session_id > 0
ORDER BY r.wait_time DESC;
```

### 3. I/O Latency per File
```sql
SELECT
    DB_NAME(fs.database_id) AS database_name,
    mf.physical_name,
    mf.type_desc AS file_type,
    fs.io_stall_read_ms / NULLIF(fs.num_of_reads, 0) AS avg_read_ms,
    fs.io_stall_write_ms / NULLIF(fs.num_of_writes, 0) AS avg_write_ms,
    fs.io_stall / NULLIF(fs.num_of_reads + fs.num_of_writes, 0) AS avg_io_ms,
    fs.num_of_reads,
    fs.num_of_writes,
    fs.io_stall_read_ms,
    fs.io_stall_write_ms
FROM sys.dm_io_virtual_file_stats(NULL, NULL) fs
JOIN sys.master_files mf ON fs.database_id = mf.database_id AND fs.file_id = mf.file_id
ORDER BY avg_io_ms DESC;
```

### 4. Memory Breakdown by Clerk
```sql
SELECT TOP 20
    type AS clerk_type,
    SUM(pages_kb) / 1024.0 AS memory_mb
FROM sys.dm_os_memory_clerks
GROUP BY type
ORDER BY SUM(pages_kb) DESC;
```

### 5. Buffer Pool Usage by Database
```sql
SELECT
    DB_NAME(database_id) AS database_name,
    COUNT(*) * 8 / 1024.0 AS buffer_pool_mb
FROM sys.dm_os_buffer_descriptors
WHERE database_id > 0
GROUP BY database_id
ORDER BY buffer_pool_mb DESC;
```

### 6. CPU Usage from Ring Buffer
```sql
SELECT TOP 30
    record.value('(./Record/SchedulerMonitorEvent/SystemHealth/ProcessUtilization)[1]', 'int') AS sql_cpu_pct,
    record.value('(./Record/SchedulerMonitorEvent/SystemHealth/SystemIdle)[1]', 'int') AS system_idle_pct,
    100 - record.value('(./Record/SchedulerMonitorEvent/SystemHealth/SystemIdle)[1]', 'int')
        - record.value('(./Record/SchedulerMonitorEvent/SystemHealth/ProcessUtilization)[1]', 'int') AS other_cpu_pct,
    DATEADD(ms, -1 * (ts_now - [timestamp]), GETDATE()) AS event_time
FROM (
    SELECT [timestamp], CONVERT(xml, record) AS record, ts_now
    FROM sys.dm_os_ring_buffers
    CROSS JOIN (SELECT ms_ticks AS ts_now FROM sys.dm_os_sys_info) AS t
    WHERE ring_buffer_type = N'RING_BUFFER_SCHEDULER_MONITOR'
      AND record LIKE '%<SystemHealth>%'
) AS ring_data
ORDER BY event_time DESC;
```

### 7. Database File Space and Auto-Growth Events
```sql
SELECT
    DB_NAME(database_id) AS database_name,
    name AS logical_name,
    physical_name,
    type_desc,
    CAST(size * 8.0 / 1024 AS DECIMAL(10,2)) AS size_mb,
    CASE max_size
        WHEN -1 THEN 'Unlimited'
        WHEN 0 THEN 'No growth'
        ELSE CAST(max_size * 8.0 / 1024 AS VARCHAR(20)) + ' MB'
    END AS max_size,
    CASE is_percent_growth
        WHEN 1 THEN CAST(growth AS VARCHAR(10)) + '%'
        ELSE CAST(growth * 8.0 / 1024 AS VARCHAR(20)) + ' MB'
    END AS growth_setting
FROM sys.master_files
ORDER BY database_id, file_id;
```

### 8. Database Status Overview
```sql
SELECT
    name,
    state_desc,
    recovery_model_desc,
    compatibility_level,
    is_auto_shrink_on,
    is_auto_close_on,
    is_auto_create_stats_on,
    is_auto_update_stats_on,
    page_verify_option_desc,
    log_reuse_wait_desc,
    DATABASEPROPERTYEX(name, 'IsAutoShrink') AS auto_shrink,
    create_date,
    collation_name
FROM sys.databases
ORDER BY name;
```

### 9. Error Log Summary (recent errors)
```sql
EXEC sys.xp_readerrorlog 0, 1, N'Error', NULL, NULL, NULL, N'desc';
-- Parameters: log number (0=current), log type (1=SQL error log), filter text
```

### 10. VLF Count per Database (log file health)
```sql
-- SQL 2016+ using sys.dm_db_log_info
SELECT
    DB_NAME(database_id) AS database_name,
    COUNT(*) AS vlf_count
FROM sys.dm_db_log_info(NULL)
GROUP BY database_id
ORDER BY vlf_count DESC;
-- Threshold: > 1000 VLFs indicates log fragmentation
```

---

## SQL Server Configuration Health Checks

```sql
-- Key configuration values
SELECT name, value_in_use, description
FROM sys.configurations
WHERE name IN (
    'max server memory (MB)',
    'min server memory (MB)',
    'max degree of parallelism',
    'cost threshold for parallelism',
    'optimize for ad hoc workloads',
    'backup compression default',
    'remote admin connections',
    'cross db ownership chaining',
    'clr enabled',
    'Ole Automation Procedures',
    'xp_cmdshell'
)
ORDER BY name;
```

---

## Key Metrics and Thresholds

| Metric | Acceptable | Warning | Critical |
|--------|-----------|---------|----------|
| Page Life Expectancy (PLE) | > 300 (legacy), > 1000 recommended | 200-300 | < 200 |
| I/O Read Latency | < 5 ms (SSD), < 20 ms (HDD) | 10-25 ms | > 25 ms |
| I/O Write Latency | < 2 ms (SSD), < 10 ms (HDD) | 5-15 ms | > 15 ms |
| CPU Usage | < 70% sustained | 70-85% | > 85% sustained |
| Signal Wait % | < 25% of total waits | 25-50% | > 50% (CPU pressure) |
| VLF Count | < 200 per database | 200-1000 | > 1000 |
| Blocking duration | < 5 seconds | 5-30 seconds | > 30 seconds |
| Log % Used | < 60% | 60-80% | > 80% |

### PLE Check
```sql
SELECT
    object_name,
    counter_name,
    cntr_value AS page_life_expectancy
FROM sys.dm_os_performance_counters
WHERE counter_name = 'Page life expectancy'
  AND object_name LIKE '%Buffer Manager%';
```

---

## Common Health Issues and Solutions

### High PAGEIOLATCH Waits
- Cause: Data not in buffer pool, disk I/O bottleneck
- Diagnose: Check I/O latency (`sys.dm_io_virtual_file_stats`), missing indexes, bad plans
- Fix: Add RAM, move to faster storage, add indexes, fix queries

### High CXPACKET / CXCONSUMER Waits
- Cause: Parallelism; one thread waiting for others
- Diagnose: Check MAXDOP, cost threshold, query plans
- Fix: Tune MAXDOP (instance and query level), increase cost threshold for parallelism (>= 25-50)

### High LCK_M_* Waits (Locking)
- Cause: Blocking, long transactions, missing indexes
- Diagnose: `sys.dm_os_waiting_tasks`, `sys.dm_tran_locks`
- Fix: Optimize transactions, add indexes on join/filter columns, consider RCSI

### High WRITELOG Waits
- Cause: Transaction log I/O bottleneck
- Diagnose: Check log file latency, VLF count, log drive
- Fix: Move log to dedicated fast disk, reduce transaction size

### Memory Pressure (low PLE)
- Cause: Insufficient RAM, memory leaks, large queries
- Diagnose: `sys.dm_os_memory_clerks`, `sys.dm_os_process_memory`, plan cache bloat
- Fix: Increase max server memory, enable optimize for ad hoc workloads, add RAM

### Autogrowth Events
- Cause: Files not pre-sized adequately
- Diagnose: Default trace or Extended Events for auto_grow
- Fix: Pre-size files, set fixed growth increments (not %), instant file initialization

---

## Automated Health Check Stored Procedure Template

```sql
CREATE OR ALTER PROCEDURE dbo.usp_SQLServerHealthCheck
AS
BEGIN
    SET NOCOUNT ON;

    -- Section 1: Instance info
    SELECT
        SERVERPROPERTY('ProductVersion') AS sql_version,
        SERVERPROPERTY('Edition') AS edition,
        SERVERPROPERTY('InstanceName') AS instance_name,
        (SELECT ms_ticks/1000/60/60 FROM sys.dm_os_sys_info) AS uptime_hours;

    -- Section 2: Databases not online
    SELECT name, state_desc FROM sys.databases WHERE state <> 0;

    -- Section 3: Low PLE
    SELECT cntr_value AS ple
    FROM sys.dm_os_performance_counters
    WHERE counter_name = 'Page life expectancy'
      AND object_name LIKE '%Buffer Manager%';

    -- Section 4: Active blocking
    SELECT COUNT(*) AS blocked_sessions
    FROM sys.dm_exec_requests WHERE blocking_session_id > 0;

    -- Section 5: High I/O latency files
    SELECT DB_NAME(database_id), physical_name,
           io_stall / NULLIF(num_of_reads + num_of_writes, 0) AS avg_io_ms
    FROM sys.dm_io_virtual_file_stats(NULL, NULL) fs
    JOIN sys.master_files mf ON fs.database_id = mf.database_id AND fs.file_id = mf.file_id
    WHERE io_stall / NULLIF(num_of_reads + num_of_writes, 0) > 25;
END;
```

---

## Extended Events for Health Monitoring

```sql
-- Create a session to capture long-running queries
CREATE EVENT SESSION [LongRunningQueries] ON SERVER
ADD EVENT sqlserver.sql_statement_completed (
    WHERE duration > 5000000  -- 5 seconds in microseconds
    ACTION(sqlserver.sql_text, sqlserver.database_name, sqlserver.username, sqlserver.plan_handle)
),
ADD EVENT sqlserver.rpc_completed (
    WHERE duration > 5000000
    ACTION(sqlserver.sql_text, sqlserver.database_name, sqlserver.username)
)
ADD TARGET package0.ring_buffer (SET max_memory = 51200)
WITH (MAX_DISPATCH_LATENCY = 30 SECONDS);

-- Start the session
ALTER EVENT SESSION [LongRunningQueries] ON SERVER STATE = START;
```

---

## References and Tools
- Glenn Berry's SQL Server Diagnostic Queries (updated regularly)
- Brent Ozar's sp_Blitz, sp_BlitzFirst, sp_BlitzCache free tools
- sys.dm_os_wait_stats reset: `DBCC SQLPERF('sys.dm_os_wait_stats', CLEAR)`
- SQL Server Error Log location: `EXEC xp_readerrorlog`
- SQL Server Agent log: `msdb.dbo.sysjobhistory`
