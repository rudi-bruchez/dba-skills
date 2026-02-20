# Diagnostic SQL Guide

A collection of essential SQL queries for diagnosing PostgreSQL health and performance.

## 1. Top 10 Queries by Total Execution Time
Requires the `pg_stat_statements` extension.
```sql
SELECT
    query,
    calls,
    total_exec_time / 1000 AS total_time_s,
    mean_exec_time AS avg_time_ms,
    rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

## 2. Identify Blocking & Locked Queries
```sql
SELECT
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocked_activity.query AS blocked_statement,
    blocking_activity.query AS blocking_statement
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_locks.pid = blocked_activity.pid
JOIN pg_catalog.pg_locks blocking_locks ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
    AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
    AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
    AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
    AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
    AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
    AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
    AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
    AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
    AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_locks.pid = blocking_activity.pid
WHERE NOT blocked_locks.granted;
```

## 3. Database Size & Table Sizes
```sql
SELECT
    datname,
    pg_size_pretty(pg_database_size(datname)) AS db_size
FROM pg_database;

-- Top 10 largest tables
SELECT
    relname AS table_name,
    pg_size_pretty(pg_total_relation_size(relid)) AS total_size,
    pg_size_pretty(pg_relation_size(relid)) AS table_size,
    pg_size_pretty(pg_total_relation_size(relid) - pg_relation_size(relid)) AS index_size
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC
LIMIT 10;
```

## 4. Transaction Throughput
```sql
SELECT
    datname,
    xact_commit AS Commits,
    xact_rollback AS Rollbacks,
    CASE WHEN (xact_commit + xact_rollback) = 0 THEN 0
         ELSE CAST(xact_rollback AS FLOAT) / (xact_commit + xact_rollback) * 100
    END AS RollbackRate
FROM pg_stat_database;
```

## 5. Check WAL Write Activity
```sql
SELECT
    pg_current_wal_lsn(),
    pg_walfile_name(pg_current_wal_lsn());
```
