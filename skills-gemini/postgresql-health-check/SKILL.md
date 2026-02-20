---
name: postgresql-health-check
description: Comprehensive PostgreSQL health check and diagnostics skill for performance monitoring, activity analysis, and database hygiene.
version: 1.0.0
tags:
  - postgresql
  - dba
  - health-check
  - diagnostics
  - performance
---

# PostgreSQL Health Check & Diagnostics

This skill provides expert guidance and actionable SQL queries for performing comprehensive health checks and performance diagnostics on PostgreSQL instances (13-17).

## Core Capabilities

- **Activity Monitoring:** Analyzing active connections, queries, and locks.
- **Resource Utilization:** Evaluating cache hits, transaction rates, and I/O.
- **Database Hygiene:** Monitoring bloat, vacuum status, and table/index stats.
- **Wait Event Analysis:** Identifying what backend processes are waiting for.

## Workflow: Periodic Health Check

1.  **Check Instance Uptime & Status:** Verify server uptime and basic configuration.
2.  **Monitor Active Connections:** Check `pg_stat_activity` for long-running queries or connection saturation.
3.  **Resource Analysis:**
    - Evaluate Cache Hit Ratio (aim for >95% for OLTP).
    - Monitor transaction throughput (commits vs. rollbacks).
4.  **Wait Events Analysis:** Identify common wait events (e.g., `IO`, `Lock`, `BufferPin`).
5.  **Database Hygiene:**
    - Check for table and index bloat.
    - Verify autovacuum status and last run times.
    - Check for missing or unused indexes.

## Diagnostic Queries

### 1. Active Connections (Non-Idle)
Identify what the database is doing *now*.
```sql
SELECT pid, usename, datname, state, wait_event_type, wait_event, query, query_start
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY query_start ASC;
```

### 2. Cache Hit Ratio
Aim for >95% for production OLTP workloads.
```sql
SELECT
    datname,
    100 * blks_hit / (blks_hit + blks_read) AS CacheHitRatio
FROM pg_stat_database
WHERE (blks_hit + blks_read) > 0;
```

### 3. Table Bloat & Vacuum Status
Identify tables that need vacuuming or have significant bloat.
```sql
SELECT
    relname AS TableName,
    n_live_tup AS LiveTuples,
    n_dead_tup AS DeadTuples,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

### 4. Index Usage & Unused Indexes
Identify indexes that are not being used but still consume resources.
```sql
SELECT
    relname AS TableName,
    indexrelname AS IndexName,
    idx_scan AS IndexScans,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
WHERE idx_scan = 0 AND schemaname = 'public'
ORDER BY relname;
```

## Best Practices (2024-2025)

- **Use `pg_stat_statements`:** Always enable this extension to track query performance across the entire instance.
- **Tune Autovacuum:** Adjust `autovacuum_vacuum_scale_factor` to be more aggressive for busy tables.
- **Use Connection Pooling:** Deploy `pgBouncer` or `pgcat` to manage a large number of client connections.
- **Monitor Replication Lag:** If using replicas, monitor `pg_stat_replication` on the primary.
- **Log Long Queries:** Set `log_min_duration_statement` to log queries exceeding a specific threshold.

## References

- [Diagnostic SQL Guide](./references/diagnostic-queries.md)
- [PG Stat Reference](./references/pg-stat-reference.md)
- [Thresholds & KPI](./references/thresholds.md)
