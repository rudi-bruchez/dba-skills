---
name: sqlserver-health-check
description: Comprehensive SQL Server health check and diagnostics skill for performance monitoring and troubleshooting.
version: 1.0.0
tags:
  - sqlserver
  - dba
  - health-check
  - diagnostics
  - performance
---

# SQL Server Health Check & Diagnostics

This skill provides expert guidance and actionable queries for performing comprehensive health checks and performance diagnostics on Microsoft SQL Server instances (2016-2025).

## Core Capabilities

- **Instance-Level Health:** Assessing server-wide configuration and resource pressure.
- **Performance Diagnostics:** Identifying bottlenecks using Wait Statistics and DMVs.
- **Resource Monitoring:** Tracking CPU, Memory, and I/O health.
- **Workload Analysis:** Identifying top resource-consuming queries and blocking.

## Workflow: Periodic Health Check

1.  **Check Server Uptime & Basics:** Verify how long the server has been running and basic configuration.
2.  **Resource Pressure Analysis:**
    - Monitor CPU utilization (aim for < 80% sustained).
    - Check Memory health via Page Life Expectancy (PLE) and Buffer Cache Hit Ratio.
    - Evaluate I/O latency (aim for < 20ms for data/log files).
3.  **Wait Statistics Analysis:** Analyze `sys.dm_os_wait_stats` to understand what SQL Server is waiting for most.
4.  **Active Request Triage:** Use `sp_WhoIsActive` or `sys.dm_exec_requests` to see what's running *now*.
5.  **Database Health:** Check for VLFs, high fragmentation, and integrity status.

## Diagnostic Queries

### 1. Wait Statistics (Top 10)
Identify what is slowing down the server globally.
```sql
SELECT TOP 10
    wait_type,
    wait_time_ms / 1000.0 AS WaitS,
    (wait_time_ms - signal_wait_time_ms) / 1000.0 AS ResourceS,
    signal_wait_time_ms / 1000.0 AS SignalS,
    max_wait_time_ms,
    percentage = CAST(100.0 * wait_time_ms / SUM(wait_time_ms) OVER() AS DECIMAL(5,2))
FROM sys.dm_os_wait_stats
WHERE wait_type NOT IN (
    'CLR_SEMAPHORE','LAZYWRITER_SLEEP','RESOURCE_QUEUE','SLEEP_TASK',
    'SLEEP_SYSTEMTASK','SQLTRACE_BUFFER_FLUSH','WAITFOR','LOGMGR_QUEUE',
    'CHECKPOINT_QUEUE','REQUEST_FOR_DEADLOCK_SEARCH','XE_TIMER_EVENT',
    'BROKER_TO_FLUSH','BROKER_TASK_STOP','CLR_MANUAL_EVENT','CLR_AUTO_EVENT',
    'DISPATCHER_QUEUE_SEMAPHORE', 'FT_IFTS_SCHEDULER_IDLE_WAIT',
    'XE_DISPATCHER_WAIT', 'XE_DISPATCHER_JOIN', 'SQLTRACE_INCREMENTAL_FLUSH_SLEEP',
    'ONDEMAND_TASK_QUEUE', 'BROKER_EVENTHANDLER', 'SLEEP_BPOOL_FLUSH', 'DIRTY_PAGE_POLL',
    'HADR_FILESTREAM_IOMGR_IOCOMPLETION', 'SP_SERVER_DIAGNOSTICS_SLEEP'
)
ORDER BY wait_time_ms DESC;
```

### 2. Memory Health (PLE)
Low Page Life Expectancy indicates memory pressure.
```sql
SELECT [object_name], [counter_name], [cntr_value]
FROM sys.dm_os_performance_counters
WHERE [counter_name] = 'Page life expectancy'
  AND [object_name] LIKE '%Manager%';
```

### 3. File I/O Latency
Identify slow disks or high I/O workloads.
```sql
SELECT
    DB_NAME(vfs.database_id) AS DatabaseName,
    df.name AS LogicalFileName,
    vfs.io_stall_read_ms / vfs.num_of_reads AS AvgReadLatency_ms,
    vfs.io_stall_write_ms / vfs.num_of_writes AS AvgWriteLatency_ms,
    vfs.io_stall / (vfs.num_of_reads + vfs.num_of_writes) AS AvgTotalLatency_ms,
    df.physical_name
FROM sys.dm_io_virtual_file_stats(NULL, NULL) AS vfs
JOIN sys.master_files AS df ON vfs.database_id = df.database_id AND vfs.file_id = df.file_id
WHERE vfs.num_of_reads > 0 AND vfs.num_of_writes > 0
ORDER BY AvgTotalLatency_ms DESC;
```

## Best Practices (2024-2025)

- **Use Query Store:** Enable Query Store on all production databases for historical performance analysis.
- **Establish Baselines:** Use `sys.dm_os_performance_counters` to capture "normal" metrics.
- **Community Tools:** Leverage `sp_WhoIsActive`, `sp_Blitz`, and Glenn Berry's Diagnostic Queries for deeper insights.
- **SQL Server 2025:** Monitor `sys.dm_os_memory_health_history` for advanced memory pressure diagnosis.

## References

- [Wait Stats Guide](./references/wait-stats-guide.md)
- [Diagnostic Queries](./references/diagnostic-queries.md)
- [Thresholds & KPI](./references/thresholds.md)
