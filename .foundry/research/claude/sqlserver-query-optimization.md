# SQL Server Query Optimization & Performance Tuning

## Overview

Query optimization involves identifying slow queries, understanding execution plans, analyzing statistics and indexes, and applying targeted fixes. The SQL Server Query Optimizer is cost-based; tuning means giving the optimizer better information or guiding it when it makes poor choices.

---

## Key DMVs for Query Performance

### Query Stats and Plan Cache
- `sys.dm_exec_query_stats` — Aggregated execution stats per cached plan (CPU, reads, duration, executions).
- `sys.dm_exec_procedure_stats` — Same as above but for stored procedures.
- `sys.dm_exec_function_stats` — For scalar user-defined functions (SQL 2016+).
- `sys.dm_exec_trigger_stats` — For triggers.
- `sys.dm_exec_cached_plans` — All cached plans with size, use count, type.
- `sys.dm_exec_sql_text(sql_handle)` — SQL text from a handle.
- `sys.dm_exec_query_plan(plan_handle)` — XML execution plan from handle.
- `sys.dm_exec_text_query_plan(plan_handle, offset, offset)` — Text plan (no XML parsing overhead).

### Query Store (SQL 2016+, highly recommended)
- `sys.query_store_query` — Queries tracked by Query Store.
- `sys.query_store_query_text` — SQL text.
- `sys.query_store_plan` — Execution plans per query.
- `sys.query_store_runtime_stats` — Runtime statistics per plan/interval.
- `sys.query_store_runtime_stats_interval` — Time intervals.
- `sys.query_store_wait_stats` — Wait stats per query/plan (SQL 2017+).

### Missing Indexes
- `sys.dm_db_missing_index_details` — Details of missing index suggestions.
- `sys.dm_db_missing_index_groups` — Groups of missing indexes.
- `sys.dm_db_missing_index_group_stats` — Impact metrics (seeks, scans, cost improvement).

### Statistics
- `sys.stats` — Statistics objects per table.
- `sys.stats_columns` — Columns in each statistics object.
- `DBCC SHOW_STATISTICS` — Histogram and density vector for a statistic.

---

## Core Diagnostic Queries

### 1. Top 20 Queries by CPU (from plan cache)
```sql
SELECT TOP 20
    qs.total_worker_time / qs.execution_count AS avg_cpu_us,
    qs.total_worker_time AS total_cpu_us,
    qs.execution_count,
    qs.total_logical_reads / qs.execution_count AS avg_logical_reads,
    qs.total_physical_reads / qs.execution_count AS avg_physical_reads,
    qs.total_elapsed_time / qs.execution_count AS avg_elapsed_us,
    qs.creation_time AS plan_compiled,
    SUBSTRING(st.text, (qs.statement_start_offset/2)+1,
        ((CASE qs.statement_end_offset WHEN -1 THEN DATALENGTH(st.text)
          ELSE qs.statement_end_offset END - qs.statement_start_offset)/2)+1) AS query_text,
    qp.query_plan
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) qp
ORDER BY qs.total_worker_time DESC;
```

### 2. Top Queries by Reads (I/O-intensive)
```sql
SELECT TOP 20
    qs.total_logical_reads / qs.execution_count AS avg_logical_reads,
    qs.total_physical_reads / qs.execution_count AS avg_physical_reads,
    qs.execution_count,
    qs.total_logical_reads,
    qs.total_worker_time / qs.execution_count AS avg_cpu_us,
    SUBSTRING(st.text, (qs.statement_start_offset/2)+1,
        ((CASE qs.statement_end_offset WHEN -1 THEN DATALENGTH(st.text)
          ELSE qs.statement_end_offset END - qs.statement_start_offset)/2)+1) AS query_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
ORDER BY qs.total_logical_reads DESC;
```

### 3. Top Queries by Duration
```sql
SELECT TOP 20
    qs.total_elapsed_time / qs.execution_count AS avg_elapsed_us,
    qs.total_elapsed_time / 1000000.0 AS total_elapsed_sec,
    qs.execution_count,
    qs.total_worker_time / qs.execution_count AS avg_cpu_us,
    SUBSTRING(st.text, (qs.statement_start_offset/2)+1,
        ((CASE qs.statement_end_offset WHEN -1 THEN DATALENGTH(st.text)
          ELSE qs.statement_end_offset END - qs.statement_start_offset)/2)+1) AS query_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
ORDER BY avg_elapsed_us DESC;
```

### 4. Missing Index Recommendations (with impact score)
```sql
SELECT TOP 20
    ROUND(migs.avg_total_user_cost * migs.avg_user_impact * (migs.user_seeks + migs.user_scans), 0) AS impact_score,
    migs.user_seeks,
    migs.user_scans,
    migs.avg_user_impact,
    migs.last_user_seek,
    DB_NAME(mid.database_id) AS database_name,
    mid.object_id,
    OBJECT_NAME(mid.object_id, mid.database_id) AS table_name,
    mid.equality_columns,
    mid.inequality_columns,
    mid.included_columns,
    'CREATE INDEX IX_' + OBJECT_NAME(mid.object_id, mid.database_id) + '_'
        + REPLACE(REPLACE(ISNULL(mid.equality_columns,'') + ISNULL('_' + mid.inequality_columns,''), ', ', '_'), '[', '') AS suggested_index
FROM sys.dm_db_missing_index_groups mig
JOIN sys.dm_db_missing_index_group_stats migs ON mig.index_group_handle = migs.group_handle
JOIN sys.dm_db_missing_index_details mid ON mig.index_handle = mid.index_handle
ORDER BY impact_score DESC;
```

### 5. Query Store: Top Regressed Queries
```sql
-- Requires Query Store enabled: ALTER DATABASE [YourDB] SET QUERY_STORE = ON
SELECT TOP 20
    qt.query_sql_text,
    q.query_id,
    p.plan_id,
    rs.avg_duration / 1000.0 AS avg_duration_ms,
    rs.avg_cpu_time / 1000.0 AS avg_cpu_ms,
    rs.avg_logical_io_reads,
    rs.count_executions,
    rs.last_execution_time
FROM sys.query_store_query q
JOIN sys.query_store_query_text qt ON q.query_text_id = qt.query_text_id
JOIN sys.query_store_plan p ON q.query_id = p.query_id
JOIN sys.query_store_runtime_stats rs ON p.plan_id = rs.plan_id
ORDER BY rs.avg_duration DESC;
```

### 6. Force a Plan in Query Store (plan regression fix)
```sql
-- First find the good plan_id and query_id from query store
EXEC sys.sp_query_store_force_plan @query_id = 42, @plan_id = 1;

-- To unforce:
EXEC sys.sp_query_store_unforce_plan @query_id = 42, @plan_id = 1;
```

### 7. Statistics Staleness Check
```sql
SELECT
    OBJECT_NAME(s.object_id) AS table_name,
    s.name AS stat_name,
    s.auto_created,
    s.user_created,
    sp.last_updated,
    sp.rows,
    sp.rows_sampled,
    CAST(sp.rows_sampled * 100.0 / NULLIF(sp.rows, 0) AS DECIMAL(5,2)) AS sample_pct,
    sp.modification_counter
FROM sys.stats s
CROSS APPLY sys.dm_db_stats_properties(s.object_id, s.stats_id) sp
WHERE OBJECT_NAME(s.object_id) IS NOT NULL
  AND sp.modification_counter > 0
ORDER BY sp.modification_counter DESC;
```

### 8. Plan Cache Size and Pollution
```sql
-- Plan cache size
SELECT
    plan_handle,
    usecounts,
    size_in_bytes / 1024.0 AS size_kb,
    cacheobjtype,
    objtype,
    SUBSTRING(st.text, 1, 200) AS sql_preview
FROM sys.dm_exec_cached_plans cp
CROSS APPLY sys.dm_exec_sql_text(cp.plan_handle) st
ORDER BY size_in_bytes DESC;

-- Single-use plans (ad hoc SQL pollution)
SELECT
    COUNT(*) AS single_use_plan_count,
    SUM(size_in_bytes) / 1024.0 / 1024.0 AS total_size_mb
FROM sys.dm_exec_cached_plans
WHERE usecounts = 1 AND objtype = 'Adhoc';

-- Fix: Enable optimize for ad hoc workloads
EXEC sp_configure 'optimize for ad hoc workloads', 1;
RECONFIGURE;
```

### 9. Implicit Conversion Detection
```sql
-- Find queries with implicit type conversions in plan cache
SELECT
    SUBSTRING(st.text, 1, 200) AS query_text,
    qp.query_plan
FROM sys.dm_exec_cached_plans cp
CROSS APPLY sys.dm_exec_sql_text(cp.plan_handle) st
CROSS APPLY sys.dm_exec_query_plan(cp.plan_handle) qp
WHERE CAST(qp.query_plan AS NVARCHAR(MAX)) LIKE '%CONVERT_IMPLICIT%';
```

### 10. Currently Running Queries with Stats
```sql
SELECT
    r.session_id,
    r.status,
    r.command,
    r.cpu_time,
    r.total_elapsed_time / 1000.0 AS elapsed_sec,
    r.logical_reads,
    r.writes,
    r.wait_type,
    r.wait_time / 1000.0 AS wait_sec,
    r.blocking_session_id,
    DB_NAME(r.database_id) AS database_name,
    SUBSTRING(st.text, (r.statement_start_offset/2)+1,
        ((CASE r.statement_end_offset WHEN -1 THEN DATALENGTH(st.text)
          ELSE r.statement_end_offset END - r.statement_start_offset)/2)+1) AS current_statement
FROM sys.dm_exec_requests r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) st
WHERE r.session_id > 50  -- Exclude system sessions
ORDER BY r.total_elapsed_time DESC;
```

---

## Execution Plan Analysis

### Reading Execution Plans
Key operators to watch for performance issues:
- **Table Scan / Clustered Index Scan** on large tables — missing indexes or insufficient WHERE clause
- **Key Lookup (Clustered)** — covering index needed, add included columns
- **Hash Match (Join)** — often indicates large data join; consider index on join key
- **Sort** — expensive on large datasets; index on ORDER BY columns if frequent
- **Parallelism (Gather Streams, Repartition Streams)** — check if parallelism is appropriate
- **Nested Loops with large outer input** — consider hash or merge join instead
- **Estimated vs. Actual rows** — large discrepancy indicates stale statistics or parameter sniffing

### Plan Comparison Commands
```sql
-- Get estimated plan only (no execution)
SET SHOWPLAN_XML ON;
GO
SELECT * FROM dbo.MyTable WHERE col = @val;
GO
SET SHOWPLAN_XML OFF;

-- Get actual plan after execution
SET STATISTICS XML ON;
GO
SELECT * FROM dbo.MyTable WHERE col = @val;
GO
SET STATISTICS XML OFF;

-- IO and time stats
SET STATISTICS IO ON;
SET STATISTICS TIME ON;
GO
SELECT * FROM dbo.MyTable WHERE col = @val;
GO
SET STATISTICS IO OFF;
SET STATISTICS TIME OFF;
```

---

## Parameter Sniffing

### Problem
SQL Server compiles a plan using the first parameter value seen (sniffed). If later calls use different parameters (high vs. low selectivity), the cached plan may be suboptimal.

### Detection
```sql
-- Find queries with high variance in execution time (parameter sniffing indicator)
SELECT
    qs.execution_count,
    qs.total_elapsed_time / qs.execution_count AS avg_elapsed_us,
    qs.max_elapsed_time AS max_elapsed_us,
    qs.min_elapsed_time AS min_elapsed_us,
    qs.max_elapsed_time - qs.min_elapsed_time AS variance_us,
    SUBSTRING(st.text, 1, 300) AS query_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
WHERE qs.max_elapsed_time > qs.min_elapsed_time * 10  -- 10x variance
ORDER BY variance_us DESC;
```

### Solutions
1. `OPTION (OPTIMIZE FOR UNKNOWN)` — use average statistics instead of sniffed value
2. `OPTION (OPTIMIZE FOR (@param = specific_value))` — optimize for known typical value
3. `OPTION (RECOMPILE)` — recompile on every execution (use sparingly)
4. `OPTION (USE HINT ('DISABLE_PARAMETER_SNIFFING'))` — SQL 2016+
5. Local variable assignment inside proc (prevents sniffing, but loses plan reuse)
6. Query Store: force a known-good plan

---

## Index Optimization for Queries

### Covering Index Pattern
```sql
-- Query: SELECT name, email FROM dbo.Customers WHERE country = 'US' AND status = 'Active'
-- Optimal index:
CREATE INDEX IX_Customers_Country_Status_Covering
ON dbo.Customers (country, status)  -- seek predicates in index key
INCLUDE (name, email);               -- projected columns as included
```

### Filtered Index (partial index)
```sql
-- For queries that always filter on a specific value
CREATE INDEX IX_Orders_Pending
ON dbo.Orders (customer_id, order_date)
INCLUDE (total_amount)
WHERE status = 'Pending';  -- Only indexes pending orders
```

### Columnstore for Analytics
```sql
-- Non-clustered columnstore on OLTP table for analytical queries
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_Sales_Analytics
ON dbo.Sales (sale_date, product_id, customer_id, amount, quantity);
```

---

## Key Query Hints and Options

```sql
-- Join hints
SELECT ... FROM A INNER LOOP JOIN B ON ...    -- Force nested loops
SELECT ... FROM A INNER HASH JOIN B ON ...    -- Force hash join
SELECT ... FROM A INNER MERGE JOIN B ON ...   -- Force merge join

-- Parallelism control
SELECT ... OPTION (MAXDOP 1);                 -- No parallelism
SELECT ... OPTION (MAXDOP 4);                 -- Use up to 4 threads

-- Recompile
SELECT ... OPTION (RECOMPILE);               -- Fresh compilation every time

-- Table hints
SELECT * FROM dbo.T WITH (NOLOCK);            -- Read uncommitted (dirty reads)
SELECT * FROM dbo.T WITH (ROWLOCK);           -- Prefer row-level locks
SELECT * FROM dbo.T WITH (PAGLOCK);           -- Prefer page locks
SELECT * FROM dbo.T WITH (INDEX(IX_Name));    -- Force specific index

-- Query Store plan forcing alternative
SELECT ... OPTION (USE PLAN N'<xml plan here>');
```

---

## Query Store Setup and Configuration

```sql
-- Enable Query Store
ALTER DATABASE [YourDatabase] SET QUERY_STORE = ON;

-- Configure Query Store
ALTER DATABASE [YourDatabase] SET QUERY_STORE (
    OPERATION_MODE = READ_WRITE,
    CLEANUP_POLICY = (STALE_QUERY_THRESHOLD_DAYS = 30),
    DATA_FLUSH_INTERVAL_SECONDS = 900,
    INTERVAL_LENGTH_MINUTES = 60,
    MAX_STORAGE_SIZE_MB = 1000,
    QUERY_CAPTURE_MODE = AUTO,  -- AUTO recommended for SQL 2019+
    SIZE_BASED_CLEANUP_MODE = AUTO,
    MAX_PLANS_PER_QUERY = 200
);

-- Force flush to disk
EXEC sys.sp_query_store_flush_db;

-- Clear all Query Store data (use with caution)
ALTER DATABASE [YourDatabase] SET QUERY_STORE CLEAR;
```

---

## Common Performance Anti-Patterns

| Anti-Pattern | Impact | Fix |
|---|---|---|
| SELECT * | Excess I/O, blocks columnstore | Select only needed columns |
| Functions in WHERE on column | Index not used | Rewrite: `WHERE col BETWEEN x AND y` instead of `WHERE YEAR(col) = 2024` |
| Implicit type conversion | Index scan instead of seek | Match data types in predicates |
| OR conditions on indexed columns | Index often not used | UNION ALL each condition |
| Non-SARGable LIKE '%value' | Always scans | Use full-text search or leading wildcard |
| Scalar UDFs in SELECT list | Row-by-row execution | Replace with inline TVFs or native code |
| Cursor over large sets | Slow, serial | Set-based operations |
| Dynamic SQL without sp_executesql | Plan cache pollution | Use sp_executesql with parameters |
| Missing covering index (key lookup) | Extra reads per row | Add INCLUDE columns |
| Stale statistics | Bad cardinality estimates | UPDATE STATISTICS or auto-update |

---

## Updating Statistics

```sql
-- Update all statistics in a database with full scan
USE [YourDatabase];
EXEC sp_updatestats;  -- Only updates stats where rows changed since last update

-- Full scan on specific table
UPDATE STATISTICS dbo.Orders WITH FULLSCAN;

-- Specific statistic
UPDATE STATISTICS dbo.Orders _WA_Sys_OrderDate WITH FULLSCAN;

-- Async stats update (SQL 2014+): auto_update_statistics_async
ALTER DATABASE [YourDatabase] SET AUTO_UPDATE_STATISTICS_ASYNC ON;
```

---

## Thresholds and Benchmarks

| Metric | Target |
|--------|--------|
| Logical reads per query | < 1000 for OLTP queries |
| Query duration (OLTP) | < 100 ms avg |
| Query duration (reporting) | Varies by acceptable SLA |
| Plan cache hit ratio | > 95% |
| Batch requests/sec | Depends on workload baseline |
| Compilations/sec | < 10% of batch requests |
| Re-compilations/sec | < 5% of compilations |
| Parameter sniffing variance | < 5x max/min ratio |
