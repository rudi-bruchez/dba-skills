---
name: sqlserver-query-optimization
description: Analyzes and optimizes slow SQL Server queries by reading execution plans, finding missing indexes, resolving parameter sniffing, and detecting implicit conversions. Use when queries are slow, consuming excessive CPU or I/O, after performance regressions, or when execution plans show warnings.
metadata:
  author: dba-skills
  version: "1.0"
  platform: SQL Server 2016+
---

# SQL Server Query Optimization

## Workflow: Find → Analyze → Fix

1. **Find** the worst-performing queries using plan cache DMVs or Query Store.
2. **Analyze** the execution plan for operators, warnings, and estimate/actual row discrepancies.
3. **Fix** with indexes, query rewrites, hints, or plan forcing.

---

## Step 1 — Find Top Slow Queries

### Top queries by CPU (plan cache)
```sql
SELECT TOP 20
    qs.total_worker_time / qs.execution_count  AS avg_cpu_us,
    qs.total_worker_time                        AS total_cpu_us,
    qs.execution_count,
    qs.total_logical_reads / qs.execution_count AS avg_logical_reads,
    qs.total_elapsed_time  / qs.execution_count AS avg_elapsed_us,
    qs.creation_time                            AS plan_compiled,
    SUBSTRING(st.text,
        (qs.statement_start_offset / 2) + 1,
        ((CASE qs.statement_end_offset WHEN -1 THEN DATALENGTH(st.text)
          ELSE qs.statement_end_offset END - qs.statement_start_offset) / 2) + 1)
        AS query_text,
    qp.query_plan
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) qp
ORDER BY qs.total_worker_time DESC;
```

### Top queries by logical reads (I/O intensive)
```sql
SELECT TOP 20
    qs.total_logical_reads / qs.execution_count AS avg_logical_reads,
    qs.total_physical_reads / qs.execution_count AS avg_physical_reads,
    qs.execution_count,
    qs.total_worker_time / qs.execution_count   AS avg_cpu_us,
    SUBSTRING(st.text,
        (qs.statement_start_offset / 2) + 1,
        ((CASE qs.statement_end_offset WHEN -1 THEN DATALENGTH(st.text)
          ELSE qs.statement_end_offset END - qs.statement_start_offset) / 2) + 1)
        AS query_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
ORDER BY qs.total_logical_reads DESC;
-- Target: < 1000 logical reads per OLTP query execution
```

### Currently running queries
```sql
SELECT
    r.session_id,
    r.status,
    r.cpu_time,
    r.total_elapsed_time / 1000.0 AS elapsed_sec,
    r.logical_reads,
    r.wait_type,
    r.wait_time / 1000.0          AS wait_sec,
    r.blocking_session_id,
    DB_NAME(r.database_id)        AS database_name,
    SUBSTRING(st.text,
        (r.statement_start_offset / 2) + 1,
        ((CASE r.statement_end_offset WHEN -1 THEN DATALENGTH(st.text)
          ELSE r.statement_end_offset END - r.statement_start_offset) / 2) + 1)
        AS current_statement
FROM sys.dm_exec_requests r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) st
WHERE r.session_id > 50
ORDER BY r.total_elapsed_time DESC;
```

---

## Step 2 — Read Execution Plans

Enable I/O and time statistics to see exactly what a query costs:
```sql
SET STATISTICS IO ON;
SET STATISTICS TIME ON;
GO
-- Run query here
GO
SET STATISTICS IO OFF;
SET STATISTICS TIME OFF;
```

**Key execution plan operators to investigate:**

| Operator | Problem Indicated | Fix |
|---|---|---|
| Table Scan / Clustered Index Scan (large table) | No useful index for predicate | Add index on filter/join columns |
| Key Lookup (Clustered) | Non-clustered index missing included columns | Add INCLUDE columns to index |
| Hash Match Join | Large, unindexed join | Add index on join key |
| Sort | No index matching ORDER BY | Add index with matching column order |
| Nested Loops with large outer input | Plan choice poor for data volume | Consider hash/merge join hints |
| Spill to TempDB warning | Insufficient memory grant | Update statistics; add/fix indexes |
| Implicit Conversion warning | Data type mismatch in predicate | Match column and parameter types |

**Warnings in the plan XML are shown as yellow bangs in SSMS.** Always investigate implicit conversions and missing index warnings.

---

## Step 3 — Missing Index Analysis

The missing index DMVs record index suggestions observed during query execution. Sort by computed impact score to prioritize.

```sql
SELECT TOP 20
    ROUND(migs.avg_total_user_cost * migs.avg_user_impact
          * (migs.user_seeks + migs.user_scans), 0)  AS impact_score,
    migs.user_seeks,
    migs.user_scans,
    migs.avg_user_impact                              AS avg_pct_improvement,
    migs.last_user_seek,
    DB_NAME(mid.database_id)                          AS database_name,
    OBJECT_NAME(mid.object_id, mid.database_id)       AS table_name,
    mid.equality_columns,
    mid.inequality_columns,
    mid.included_columns,
    'CREATE NONCLUSTERED INDEX IX_'
        + OBJECT_NAME(mid.object_id, mid.database_id)
        + '_' + CONVERT(VARCHAR(10), mig.index_group_handle)
        + ' ON ' + mid.statement
        + ' (' + ISNULL(mid.equality_columns, '')
        + CASE WHEN mid.inequality_columns IS NOT NULL
               THEN CASE WHEN mid.equality_columns IS NOT NULL THEN ', ' ELSE '' END
                    + mid.inequality_columns ELSE '' END + ')'
        + ISNULL(' INCLUDE (' + mid.included_columns + ')', '')
        AS suggested_create_statement
FROM sys.dm_db_missing_index_details mid
JOIN sys.dm_db_missing_index_groups mig
    ON mid.index_handle = mig.index_handle
JOIN sys.dm_db_missing_index_group_stats migs
    ON mig.index_group_handle = migs.group_handle
ORDER BY impact_score DESC;
```

**Do not blindly create every suggested index.** Each non-clustered index adds write overhead. Evaluate:
- Is impact_score > 100,000?
- Does the suggestion overlap with an existing index (add INCLUDE instead)?
- Are equality columns listed first, inequality columns last?

---

## Step 4 — Parameter Sniffing Detection and Fixes

Parameter sniffing occurs when SQL Server compiles a plan for the first parameter value seen. Later calls with different parameter distributions use the same plan, which may be inefficient.

### Detection: high variance in min/max elapsed time
```sql
SELECT
    qs.execution_count,
    qs.total_elapsed_time / qs.execution_count AS avg_elapsed_us,
    qs.max_elapsed_time                         AS max_elapsed_us,
    qs.min_elapsed_time                         AS min_elapsed_us,
    qs.max_elapsed_time - qs.min_elapsed_time   AS variance_us,
    SUBSTRING(st.text, 1, 300)                  AS query_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
WHERE qs.max_elapsed_time > qs.min_elapsed_time * 10  -- 10x variance
ORDER BY variance_us DESC;
```

### Fix options (in order of preference)

1. **OPTIMIZE FOR UNKNOWN** — use average statistics, not sniffed value:
   ```sql
   SELECT * FROM dbo.Orders WHERE customer_id = @cust_id
   OPTION (OPTIMIZE FOR (@cust_id UNKNOWN));
   ```

2. **OPTIMIZE FOR specific value** — optimize for the typical high-volume case:
   ```sql
   OPTION (OPTIMIZE FOR (@cust_id = 1001));
   ```

3. **Disable parameter sniffing for the statement** (SQL 2016+):
   ```sql
   OPTION (USE HINT ('DISABLE_PARAMETER_SNIFFING'));
   ```

4. **RECOMPILE** — fresh compilation every call (use only for infrequent, expensive queries):
   ```sql
   OPTION (RECOMPILE);
   ```

5. **Force a known-good plan via Query Store:**
   ```sql
   EXEC sys.sp_query_store_force_plan @query_id = 42, @plan_id = 7;
   ```

---

## Step 5 — Implicit Conversion Detection

Implicit conversions cause index scans instead of seeks. They are invisible in the query text but appear as `CONVERT_IMPLICIT` in the execution plan XML.

### Find queries with implicit conversions in plan cache
```sql
SELECT
    SUBSTRING(st.text, 1, 300) AS query_text,
    qp.query_plan
FROM sys.dm_exec_cached_plans cp
CROSS APPLY sys.dm_exec_sql_text(cp.plan_handle) st
CROSS APPLY sys.dm_exec_query_plan(cp.plan_handle) qp
WHERE CAST(qp.query_plan AS NVARCHAR(MAX)) LIKE '%CONVERT_IMPLICIT%';
```

**Common causes and fixes:**

| Scenario | Cause | Fix |
|---|---|---|
| `WHERE varchar_col = @nvarchar_param` | Type mismatch | Change parameter to `VARCHAR` or alter column to `NVARCHAR` |
| `WHERE int_col = @varchar_param` | Application sends wrong type | Fix ORM/app data type binding |
| `WHERE date_col = @string` | String compared to date | Use `CONVERT(DATE, @string)` or fix binding |

---

## Step 6 — Query Store for Regression Detection

Query Store persists execution plans and statistics across restarts. Use it to identify and fix plan regressions without code changes.

### Enable Query Store
```sql
ALTER DATABASE [YourDatabase] SET QUERY_STORE = ON;
ALTER DATABASE [YourDatabase] SET QUERY_STORE (
    OPERATION_MODE      = READ_WRITE,
    CLEANUP_POLICY      = (STALE_QUERY_THRESHOLD_DAYS = 30),
    DATA_FLUSH_INTERVAL_SECONDS = 900,
    INTERVAL_LENGTH_MINUTES     = 60,
    MAX_STORAGE_SIZE_MB         = 1000,
    QUERY_CAPTURE_MODE          = AUTO,
    SIZE_BASED_CLEANUP_MODE     = AUTO
);
```

### Find regressed queries (high avg duration)
```sql
SELECT TOP 20
    qt.query_sql_text,
    q.query_id,
    p.plan_id,
    rs.avg_duration / 1000.0       AS avg_duration_ms,
    rs.avg_cpu_time / 1000.0       AS avg_cpu_ms,
    rs.avg_logical_io_reads,
    rs.count_executions,
    rs.last_execution_time
FROM sys.query_store_query q
JOIN sys.query_store_query_text qt  ON q.query_text_id  = qt.query_text_id
JOIN sys.query_store_plan p         ON q.query_id       = p.query_id
JOIN sys.query_store_runtime_stats rs ON p.plan_id      = rs.plan_id
ORDER BY rs.avg_duration DESC;
```

### Force a known-good plan
```sql
-- Force plan (no code change required)
EXEC sys.sp_query_store_force_plan @query_id = 42, @plan_id = 1;

-- Unforce
EXEC sys.sp_query_store_unforce_plan @query_id = 42, @plan_id = 1;
```

---

## Common Anti-Patterns Quick Reference

| Anti-Pattern | Impact | Fix |
|---|---|---|
| `SELECT *` | Excess I/O; blocks columnstore | Select only needed columns |
| Function on column in WHERE: `WHERE YEAR(col) = 2024` | Index scan | Rewrite: `WHERE col >= '2024-01-01' AND col < '2025-01-01'` |
| Implicit type conversion | Index scan | Match data types |
| `OR` on indexed columns | Index often skipped | Rewrite as `UNION ALL` |
| `LIKE '%value'` leading wildcard | Always scans | Use full-text search |
| Scalar UDF in SELECT list | Row-by-row execution | Replace with inline TVF |
| Dynamic SQL without `sp_executesql` | Plan cache pollution | Parameterize with `sp_executesql` |
| Missing covering index (key lookup) | Extra reads per row | Add INCLUDE columns |

---

## Statistics Health Check

Stale statistics cause poor cardinality estimates, which lead to bad plan choices.

```sql
SELECT
    OBJECT_NAME(s.object_id) AS table_name,
    s.name                   AS stat_name,
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

-- Force full-scan update on a high-modification table
UPDATE STATISTICS dbo.Orders WITH FULLSCAN;
```

---

## References

- [Execution plan guide](references/execution-plan-guide.md) — key plan operators and what they mean
- [Index strategies](references/index-strategies.md) — covering indexes, included columns, filtered indexes
- [Anti-patterns](references/antipatterns.md) — common T-SQL anti-patterns with before/after fixes
- [Examples](examples/examples.md) — slow query analysis, missing index, parameter sniffing, implicit conversion, regression
