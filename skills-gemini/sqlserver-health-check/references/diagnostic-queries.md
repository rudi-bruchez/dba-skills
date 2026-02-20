# Diagnostic Queries

Use these queries for a deeper dive into SQL Server health.

## 1. Top CPU-Consuming Queries
Identify queries putting high pressure on the CPU.
```sql
SELECT TOP 10
    SUBSTRING(st.text, (qs.statement_start_offset/2) + 1,
    ((CASE qs.statement_end_offset WHEN -1 THEN DATALENGTH(st.text) ELSE qs.statement_end_offset END
        - qs.statement_start_offset)/2) + 1) AS QueryText,
    qs.total_worker_time / 1000 AS TotalCPUTime_ms,
    qs.execution_count,
    qs.total_worker_time / qs.execution_count / 1000 AS AvgCPUTime_ms,
    qp.query_plan
FROM sys.dm_exec_query_stats AS qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) AS st
CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) AS qp
ORDER BY qs.total_worker_time DESC;
```

## 2. Active Blocking & Lead Blockers
Identify sessions preventing others from proceeding.
```sql
SELECT
    blocking_session_id AS LeadBlocker,
    session_id AS BlockedSession,
    wait_type,
    wait_time / 1000 AS WaitTime_s,
    last_wait_type,
    text AS QueryText
FROM sys.dm_exec_requests
CROSS APPLY sys.dm_exec_sql_text(sql_handle)
WHERE blocking_session_id <> 0;
```

## 3. Buffer Cache Hit Ratio
Percentage of data found in memory. Aim for >95% for OLTP workloads.
```sql
SELECT
    (a.cntr_value * 1.0 / b.cntr_value) * 100.0 AS BufferCacheHitRatio
FROM sys.dm_os_performance_counters  a
JOIN  sys.dm_os_performance_counters  b ON  a.object_name = b.object_name
WHERE a.counter_name = 'Buffer cache hit ratio'
  AND b.counter_name = 'Buffer cache hit ratio base'
  AND a.object_name LIKE '%Buffer Manager%';
```

## 4. Database Integrity Status
Check the last time `DBCC CHECKDB` was run.
```sql
SELECT
    name AS DatabaseName,
    DATABASEPROPERTYEX(name, 'LastGoodCheckDbTime') AS LastGoodCheckDbTime
FROM sys.databases;
```

## 5. SQL Server Configuration (Advanced)
Check for `MAXDOP` and `Cost Threshold for Parallelism`.
```sql
SELECT name, value, value_in_use, minimum, maximum, [description]
FROM sys.configurations
WHERE name IN ('max degree of parallelism', 'cost threshold for parallelism', 'optimize for ad hoc workloads');
```
