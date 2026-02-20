---
name: postgresql-query-optimization
description: Specialized skill for optimizing PostgreSQL query performance, analyzing execution plans, and implementing effective indexing strategies.
version: 1.0.0
tags:
  - postgresql
  - dba
  - query-tuning
  - explain
  - optimization
---

# PostgreSQL Query Optimization

This skill provides expert techniques for analyzing, tuning, and optimizing PostgreSQL queries to reduce resource consumption and improve execution speed.

## Core Capabilities

- **Execution Plan Analysis:** Interpreting `EXPLAIN (ANALYZE, BUFFERS)` output.
- **Indexing Strategies:** Designing B-tree, GIN, GiST, and BRIN indexes.
- **SARGability Tuning:** Rewriting queries to use indexes efficiently.
- **Configuration Tuning:** Optimizing server parameters (`work_mem`, `shared_buffers`).
- **Partitioning Strategy:** Implementing declarative partitioning for large tables.

## Workflow: Query Tuning

1.  **Identify Slow Queries:** Use `pg_stat_statements` to find top resource-consuming queries.
2.  **Analyze the Execution Plan:** Run `EXPLAIN (ANALYZE, BUFFERS)` on the query.
3.  **Identify Bottlenecks:** Look for `Seq Scan`, `Hash Join`, and `Sort` operations on large data sets.
4.  **Evaluate Indexing:**
    - Is an index missing on a `WHERE` or `JOIN` column?
    - Can a `GIN` index optimize a `JSONB` or array search?
5.  **Check Visibility Map:** Ensure autovacuum is active to enable "Index Only Scans."
6.  **Review Statistics:** Ensure statistics are up-to-date (`ANALYZE`).
7.  **Optimize Server Parameters:** Adjust `work_mem` for queries with large sorts or hash joins.

## Optimization Techniques

### 1. Identify Top Queries by Total Time
```sql
SELECT query, calls, total_exec_time, mean_exec_time, rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

### 2. Analyze a Query Plan
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE customer_id = 12345;
```

### 3. Identify Sequential Scans on Large Tables
```sql
SELECT relname AS table_name, seq_scan, seq_tup_read, idx_scan, idx_tup_fetch
FROM pg_stat_user_tables
WHERE seq_scan > 0 AND n_live_tup > 100000
ORDER BY seq_tup_read DESC;
```

### 4. Create a GIN Index for JSONB
```sql
CREATE INDEX idx_user_data_gin ON users USING GIN (data jsonb_path_ops);
```

## Best Practices (2024-2025)

- **Use `EXPLAIN (ANALYZE, BUFFERS)`:** Always include `BUFFERS` to see memory/disk usage.
- **Avoid Anti-Patterns:** Stop using `SELECT *`, `NOT IN`, and functions on indexed columns (e.g., `WHERE LOWER(email) = 'rudi'`).
- **Leverage Partitioning:** Use declarative partitioning for tables > 100 GB.
- **Parallel Query:** Tune `max_parallel_workers_per_gather` for multi-core performance.
- **Index Only Scans:** Aim for these by keeping the visibility map clean via autovacuum.

## References

- [Explain Guide](./references/explain-guide.md)
- [Index Types](./references/index-types.md)
- [Planner Configuration](./references/planner-config.md)
