---
name: sqlserver-health-check
description: Diagnoses SQL Server instance health by analyzing DMVs, wait statistics, blocking chains, memory pressure, CPU usage, I/O stalls, and database integrity. Use when SQL Server is slow, unresponsive, experiencing errors, or when performing routine health audits.
metadata:
  author: dba-skills
  version: "1.0"
  platform: SQL Server 2016+
---

# SQL Server Health Check

## Quick Health Overview Workflow

Run these five steps in sequence to establish a health baseline before diving into specifics.

**Step 1 — Instance vitals**
```sql
SELECT
    SERVERPROPERTY('ProductVersion')  AS sql_version,
    SERVERPROPERTY('Edition')         AS edition,
    SERVERPROPERTY('InstanceName')    AS instance_name,
    (SELECT ms_ticks / 1000 / 60 / 60 FROM sys.dm_os_sys_info) AS uptime_hours;
```

**Step 2 — Databases not ONLINE**
```sql
SELECT name, state_desc, log_reuse_wait_desc
FROM sys.databases
WHERE state <> 0;
```

**Step 3 — Page Life Expectancy (memory pressure signal)**
```sql
SELECT cntr_value AS page_life_expectancy
FROM sys.dm_os_performance_counters
WHERE counter_name = 'Page life expectancy'
  AND object_name LIKE '%Buffer Manager%';
-- Healthy: > 300 (legacy). Modern guidance: > 1000. Critical: < 200.
```

**Step 4 — Active blocking count**
```sql
SELECT COUNT(*) AS blocked_sessions
FROM sys.dm_exec_requests
WHERE blocking_session_id > 0;
```

**Step 5 — I/O files with high latency**
```sql
SELECT DB_NAME(fs.database_id) AS database_name,
       mf.physical_name,
       fs.io_stall / NULLIF(fs.num_of_reads + fs.num_of_writes, 0) AS avg_io_ms
FROM sys.dm_io_virtual_file_stats(NULL, NULL) fs
JOIN sys.master_files mf
    ON fs.database_id = mf.database_id AND fs.file_id = mf.file_id
WHERE fs.io_stall / NULLIF(fs.num_of_reads + fs.num_of_writes, 0) > 25
ORDER BY avg_io_ms DESC;
```

---

## Wait Statistics Analysis

Wait statistics are the most reliable signal for diagnosing the root cause of SQL Server slowdowns. Query `sys.dm_os_wait_stats` after excluding benign idle waits.

```sql
SELECT TOP 20
    wait_type,
    wait_time_ms / 1000.0                              AS wait_time_sec,
    (wait_time_ms - signal_wait_time_ms) / 1000.0     AS resource_wait_sec,
    signal_wait_time_ms / 1000.0                       AS signal_wait_sec,
    waiting_tasks_count,
    CASE WHEN waiting_tasks_count = 0 THEN 0
         ELSE wait_time_ms / waiting_tasks_count
    END AS avg_wait_ms
FROM sys.dm_os_wait_stats
WHERE wait_type NOT IN (
    'SLEEP_TASK','BROKER_TO_FLUSH','BROKER_TASK_STOP','CLR_AUTO_EVENT',
    'DISPATCHER_QUEUE_SEMAPHORE','FT_IFTS_SCHEDULER_IDLE_WAIT',
    'LAZYWRITER_SLEEP','LOGMGR_QUEUE','ONDEMAND_TASK_QUEUE',
    'REQUEST_FOR_DEADLOCK_SEARCH','RESOURCE_QUEUE','SERVER_IDLE_CHECK',
    'SLEEP_DBSTARTUP','SLEEP_DCOMSTARTUP','SLEEP_MASTERDBREADY',
    'SLEEP_MASTERMDREADY','SLEEP_MASTERUPGRADED','SLEEP_MSDBSTARTUP',
    'SLEEP_SYSTEMTASK','SLEEP_TEMPDBSTARTUP','SNI_HTTP_ACCEPT',
    'SP_SERVER_DIAGNOSTICS_SLEEP','SQLTRACE_BUFFER_FLUSH',
    'SQLTRACE_INCREMENTAL_FLUSH_SLEEP','WAITFOR','XE_DISPATCHER_WAIT',
    'XE_TIMER_EVENT','BROKER_EVENTHANDLER','CHECKPOINT_QUEUE',
    'DBMIRROR_EVENTS_QUEUE','SQLTRACE_WAIT_ENTRIES',
    'WAIT_XTP_OFFLINE_CKPT_NEW_LOG','HADR_WORK_QUEUE','HADR_FILESTREAM_IOMGR_IOCOMPLETION'
)
ORDER BY wait_time_ms DESC;
```

**Key wait type interpretation:**

| Wait Type | Root Cause | First Action |
|---|---|---|
| PAGEIOLATCH_SH/EX | Missing indexes, disk I/O slow, insufficient RAM | Check I/O latency, add indexes |
| LCK_M_* | Blocking, long transactions | See Blocking section |
| CXPACKET / CXCONSUMER | Parallelism imbalance | Tune MAXDOP, increase cost threshold |
| WRITELOG | Transaction log I/O bottleneck | Move log to fast dedicated disk |
| SOS_SCHEDULER_YIELD | CPU pressure, spinlock contention | Check CPU usage, reduce parallelism |
| RESOURCE_SEMAPHORE | Memory grant queuing | Tune max server memory, fix queries |

Signal wait > 25% of total waits indicates CPU pressure.

See [wait stats guide](references/wait-stats-guide.md) for the full list of wait types.

---

## Blocking and Deadlock Detection

### Current blocking chains
```sql
SELECT
    r.session_id,
    r.blocking_session_id,
    r.wait_type,
    r.wait_time / 1000.0   AS wait_sec,
    r.status,
    DB_NAME(r.database_id) AS database_name,
    SUBSTRING(st.text,
        (r.statement_start_offset / 2) + 1,
        ((CASE r.statement_end_offset
              WHEN -1 THEN DATALENGTH(st.text)
              ELSE r.statement_end_offset
          END - r.statement_start_offset) / 2) + 1) AS current_statement,
    s.login_name,
    s.host_name,
    s.program_name
FROM sys.dm_exec_requests r
JOIN sys.dm_exec_sessions s ON r.session_id = s.session_id
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) st
WHERE r.blocking_session_id > 0
ORDER BY r.wait_time DESC;
```

### Find the head blocker (the session blocking all others)
```sql
SELECT r.blocking_session_id AS head_blocker,
       s.login_name, s.host_name, s.program_name,
       s.open_transaction_count,
       SUBSTRING(st.text, 1, 500) AS head_blocker_sql
FROM sys.dm_exec_sessions s
LEFT JOIN sys.dm_exec_requests r ON s.session_id = r.session_id
CROSS APPLY sys.dm_exec_sql_text(COALESCE(r.sql_handle, s.sql_handle)) st
WHERE s.session_id IN (
    SELECT DISTINCT blocking_session_id
    FROM sys.dm_exec_requests
    WHERE blocking_session_id > 0
)
  AND s.session_id NOT IN (
    SELECT session_id FROM sys.dm_exec_requests WHERE blocking_session_id > 0
);
```

### Read deadlock graphs (system_health XE session)
```sql
SELECT
    xdr.value('@timestamp', 'datetime2') AS deadlock_time,
    xdr.query('.') AS deadlock_graph
FROM (
    SELECT CAST(target_data AS XML) AS target_data
    FROM sys.dm_xe_session_targets t
    JOIN sys.dm_xe_sessions s ON t.event_session_address = s.address
    WHERE s.name = 'system_health' AND t.target_name = 'ring_buffer'
) AS data
CROSS APPLY target_data.nodes('//RingBufferTarget/event[@name="xml_deadlock_report"]') AS xdt(xdr)
ORDER BY deadlock_time DESC;
```

**Resolution steps:**
1. Identify the head blocker session and its `open_transaction_count`.
2. Check `program_name` and `host_name` to identify the application.
3. If blocking duration > 30 seconds in production, consider `KILL <session_id>` for the head blocker.
4. Long-term fix: add indexes on join/filter columns, shorten transactions, enable Read Committed Snapshot Isolation (RCSI).

---

## Memory Pressure

### PLE and memory clerk breakdown
```sql
-- Memory allocation by clerk type (top consumers)
SELECT TOP 20
    type AS clerk_type,
    SUM(pages_kb) / 1024.0 AS memory_mb
FROM sys.dm_os_memory_clerks
GROUP BY type
ORDER BY SUM(pages_kb) DESC;

-- Buffer pool by database
SELECT
    DB_NAME(database_id) AS database_name,
    COUNT(*) * 8 / 1024.0 AS buffer_pool_mb
FROM sys.dm_os_buffer_descriptors
WHERE database_id > 0
GROUP BY database_id
ORDER BY buffer_pool_mb DESC;

-- Single-use plan cache pollution
SELECT
    COUNT(*) AS single_use_plan_count,
    SUM(size_in_bytes) / 1024.0 / 1024.0 AS total_size_mb
FROM sys.dm_exec_cached_plans
WHERE usecounts = 1 AND objtype = 'Adhoc';
-- If > 100 MB: enable "optimize for ad hoc workloads"
```

**Memory health thresholds:**
- PLE > 1000: healthy
- PLE 300–1000: monitor
- PLE < 300: likely memory pressure — check max server memory, add RAM

**Quick fix for plan cache pollution:**
```sql
EXEC sp_configure 'optimize for ad hoc workloads', 1;
RECONFIGURE;
```

---

## CPU and I/O Top Consumers

### CPU usage from ring buffer (last 30 samples)
```sql
SELECT TOP 30
    record.value('(./Record/SchedulerMonitorEvent/SystemHealth/ProcessUtilization)[1]', 'int')
        AS sql_cpu_pct,
    record.value('(./Record/SchedulerMonitorEvent/SystemHealth/SystemIdle)[1]', 'int')
        AS system_idle_pct,
    100 - record.value('(./Record/SchedulerMonitorEvent/SystemHealth/SystemIdle)[1]', 'int')
        - record.value('(./Record/SchedulerMonitorEvent/SystemHealth/ProcessUtilization)[1]', 'int')
        AS other_process_cpu_pct,
    DATEADD(ms, -1 * (ts_now - [timestamp]), GETDATE()) AS event_time
FROM (
    SELECT [timestamp],
           CONVERT(xml, record) AS record,
           ts_now
    FROM sys.dm_os_ring_buffers
    CROSS JOIN (SELECT ms_ticks AS ts_now FROM sys.dm_os_sys_info) t
    WHERE ring_buffer_type = N'RING_BUFFER_SCHEDULER_MONITOR'
      AND record LIKE '%<SystemHealth>%'
) AS ring_data
ORDER BY event_time DESC;
-- Warning: sql_cpu_pct > 85% sustained. Check top query consumers next.
```

### Top queries by CPU (from plan cache)
```sql
SELECT TOP 10
    qs.total_worker_time / qs.execution_count  AS avg_cpu_us,
    qs.total_worker_time                        AS total_cpu_us,
    qs.execution_count,
    qs.total_logical_reads / qs.execution_count AS avg_logical_reads,
    SUBSTRING(st.text,
        (qs.statement_start_offset / 2) + 1,
        ((CASE qs.statement_end_offset WHEN -1 THEN DATALENGTH(st.text)
          ELSE qs.statement_end_offset END - qs.statement_start_offset) / 2) + 1)
        AS query_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
ORDER BY qs.total_worker_time DESC;
```

### I/O latency per file
```sql
SELECT
    DB_NAME(fs.database_id)              AS database_name,
    mf.physical_name,
    mf.type_desc                         AS file_type,
    fs.io_stall_read_ms  / NULLIF(fs.num_of_reads, 0)           AS avg_read_ms,
    fs.io_stall_write_ms / NULLIF(fs.num_of_writes, 0)          AS avg_write_ms,
    fs.io_stall          / NULLIF(fs.num_of_reads + fs.num_of_writes, 0) AS avg_io_ms,
    fs.num_of_reads,
    fs.num_of_writes
FROM sys.dm_io_virtual_file_stats(NULL, NULL) fs
JOIN sys.master_files mf
    ON fs.database_id = mf.database_id AND fs.file_id = mf.file_id
ORDER BY avg_io_ms DESC;
-- Read: > 5 ms (SSD) or > 20 ms (HDD) = warning. Write: > 2 ms (SSD) or > 10 ms (HDD) = warning.
```

---

## Database State Checks

```sql
-- Database status, settings, and VLF count
SELECT
    d.name,
    d.state_desc,
    d.recovery_model_desc,
    d.compatibility_level,
    d.is_auto_shrink_on,          -- Should be 0 (OFF)
    d.is_auto_close_on,           -- Should be 0 (OFF)
    d.page_verify_option_desc,    -- Should be CHECKSUM
    d.log_reuse_wait_desc,
    vlf.vlf_count
FROM sys.databases d
LEFT JOIN (
    SELECT database_id, COUNT(*) AS vlf_count
    FROM sys.dm_db_log_info(NULL)
    GROUP BY database_id
) vlf ON d.database_id = vlf.database_id
ORDER BY d.name;
-- VLF count > 1000: log fragmentation — shrink log then grow with fixed increments
```

**Configuration health check:**
```sql
SELECT name, value_in_use, description
FROM sys.configurations
WHERE name IN (
    'max server memory (MB)', 'min server memory (MB)',
    'max degree of parallelism', 'cost threshold for parallelism',
    'optimize for ad hoc workloads', 'backup compression default',
    'xp_cmdshell', 'clr enabled', 'Ole Automation Procedures'
)
ORDER BY name;
```

---

## Health Metric Thresholds

See [thresholds reference](references/thresholds.md) for a full table with warning/critical levels and remediation steps.

| Metric | Healthy | Warning | Critical |
|---|---|---|---|
| Page Life Expectancy | > 1000 | 300–1000 | < 300 |
| I/O Read Latency (SSD) | < 5 ms | 5–10 ms | > 10 ms |
| I/O Write Latency (SSD) | < 2 ms | 2–5 ms | > 5 ms |
| CPU sustained | < 70% | 70–85% | > 85% |
| Signal wait % | < 25% | 25–50% | > 50% |
| VLF count | < 200 | 200–1000 | > 1000 |
| Blocking duration | < 5 sec | 5–30 sec | > 30 sec |

---

## References

- [Diagnostic queries](references/diagnostic-queries.md) — complete T-SQL scripts for all health checks
- [Wait stats guide](references/wait-stats-guide.md) — common wait types, meanings, and remediation
- [Thresholds](references/thresholds.md) — all health metric thresholds with recommended actions
- [Examples](examples/examples.md) — slow server, blocking, memory pressure, disk I/O scenarios
