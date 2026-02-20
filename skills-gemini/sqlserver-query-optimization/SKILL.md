---
name: sqlserver-query-optimization
description: Specialized skill for optimizing T-SQL query performance and execution plans in Microsoft SQL Server.
version: 1.0.0
tags:
  - sqlserver
  - dba
  - query-tuning
  - execution-plan
  - optimization
---

# SQL Server Query Optimization

This skill provides expert techniques for analyzing, tuning, and optimizing SQL Server queries to reduce resource consumption and improve execution speed.

## Core Capabilities

- **Execution Plan Analysis:** Interpreting graphical and XML execution plans.
- **Indexing Strategies:** Designing covering, filtered, and columnstore indexes.
- **SARGability Tuning:** Rewriting queries to use indexes efficiently.
- **Statistics Management:** Ensuring the Optimizer has accurate row counts.
- **Intelligent Query Processing:** Leveraging modern SQL Server features (2019-2025).

## Workflow: Query Tuning

1.  **Identify Slow Queries:** Use Query Store or `sys.dm_exec_query_stats` to find top resource-consuming queries.
2.  **Analyze the Execution Plan:** Look for "expensive" operators (Scans, Sorts, Hash Joins) and high-cost warnings.
3.  **Check for SARGability Issues:** Verify that `WHERE` and `JOIN` clauses aren't using functions on indexed columns.
4.  **Evaluate Indexing:**
    - Are there "Missing Index" recommendations?
    - Can a "Covering Index" eliminate a `Key Lookup`?
5.  **Review Statistics:** Ensure statistics are up-to-date (`AUTO_UPDATE_STATISTICS`) and accurate (`WITH FULLSCAN`).
6.  **Apply Optimization Hints:** Use `OPTION (RECOMPILE)`, `MAXDOP`, or `Query Store Hints` as a last resort.

## Optimization Techniques

### 1. Identify Top Queries by CPU
```sql
SELECT TOP 20
    st.text AS QueryText,
    qs.total_worker_time / 1000 AS TotalCPUTime_ms,
    qs.execution_count,
    qs.total_worker_time / qs.execution_count / 1000 AS AvgCPUTime_ms,
    qp.query_plan
FROM sys.dm_exec_query_stats AS qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) AS st
CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) AS qp
ORDER BY qs.total_worker_time DESC;
```

### 2. Identify Top Queries by I/O
```sql
SELECT TOP 20
    st.text AS QueryText,
    qs.total_logical_reads AS TotalReads,
    qs.total_logical_writes AS TotalWrites,
    qs.execution_count,
    qp.query_plan
FROM sys.dm_exec_query_stats AS qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) AS st
CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) AS qp
ORDER BY qs.total_logical_reads DESC;
```

### 3. Check for Missing Indexes
```sql
SELECT TOP 10
    id.statement AS TableName,
    id.equality_columns,
    id.inequality_columns,
    id.included_columns,
    gs.avg_user_impact AS PotentialImpact,
    gs.avg_total_user_cost * gs.avg_user_impact * (gs.user_seeks + gs.user_scans) AS TotalBenefit
FROM sys.dm_db_missing_index_group_stats AS gs
JOIN sys.dm_db_missing_index_groups AS ig ON gs.group_handle = ig.index_group_handle
JOIN sys.dm_db_missing_index_details AS id ON ig.index_handle = id.index_handle
ORDER BY TotalBenefit DESC;
```

## Best Practices (2024-2025)

- **Prefer Query Store:** Always use Query Store to track execution plan changes and historical performance.
- **Avoid Anti-Patterns:** Stop using `SELECT *`, `NOT IN`, and functions on indexed columns (e.g., `WHERE YEAR(Date) = 2023`).
- **Use IQP Features:** Leverage "Memory Grant Feedback" and "Adaptive Joins" in SQL Server 2019+.
- **Resumable Index Operations:** Use resumable operations for large index builds and rebuilds to avoid maintenance windows.

## References

- [Execution Plan Guide](./references/execution-plan-guide.md)
- [Indexing Strategies](./references/indexing-strategies.md)
- [Query Tuning Anti-patterns](./references/antipatterns.md)
