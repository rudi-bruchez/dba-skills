# PostgreSQL Health Check & Diagnostics

## Overview
PostgreSQL exposes runtime statistics through a rich set of system catalog views prefixed with `pg_stat_*` and `pg_statio_*`. These are the primary tools for real-time health assessment. Health checks fall into five categories: connectivity, activity, I/O, replication, and resource usage.

---

## Key System Catalog Views

### Activity & Connections
| View | Purpose |
|------|---------|
| `pg_stat_activity` | Current sessions, queries, wait events, state |
| `pg_stat_database` | Per-database stats: commits, rollbacks, cache hits |
| `pg_stat_user_tables` | Per-table stats: seq/idx scans, tuples, vacuums |
| `pg_stat_user_indexes` | Per-index usage stats |
| `pg_statio_user_tables` | Buffer hits vs disk reads per table |
| `pg_statio_user_indexes` | Buffer hits vs disk reads per index |
| `pg_stat_bgwriter` | Background writer and checkpoint stats |
| `pg_stat_wal` | WAL generation and write stats (PG14+) |
| `pg_stat_replication` | Streaming replication lag and state |
| `pg_stat_statements` | Aggregated query execution stats (extension) |
| `pg_locks` | Lock information |
| `pg_stat_progress_vacuum` | Live vacuum progress |
| `pg_stat_progress_analyze` | Live analyze progress |
| `pg_stat_progress_index` | Live index build progress |

### Key Functions
- `pg_postmaster_start_time()` — when the server started
- `pg_database_size(dbname)` — database size in bytes
- `pg_relation_size(relation)` / `pg_total_relation_size(relation)` — table/index size
- `pg_size_pretty(bytes)` — human-readable size
- `pg_current_wal_lsn()` / `pg_walfile_name()` — current WAL position
- `pg_stat_reset()` — reset per-database stats
- `pg_stat_reset_single_table_counters(oid)` — reset one table's stats
- `now() - pg_postmaster_start_time()` — server uptime

---

## Diagnostic SQL Queries

### 1. Server Uptime & Version
```sql
SELECT version();
SELECT pg_postmaster_start_time(),
       now() - pg_postmaster_start_time() AS uptime;
```

### 2. Active Connections Summary
```sql
SELECT count(*),
       state,
       wait_event_type,
       wait_event
FROM pg_stat_activity
WHERE pid <> pg_backend_pid()
GROUP BY state, wait_event_type, wait_event
ORDER BY count DESC;
```

### 3. Max Connections Usage
```sql
SELECT count(*) AS current_connections,
       (SELECT setting::int FROM pg_settings WHERE name = 'max_connections') AS max_connections,
       round(100.0 * count(*) /
         (SELECT setting::int FROM pg_settings WHERE name = 'max_connections'), 1) AS pct_used
FROM pg_stat_activity;
```
**Threshold:** Alert when connections exceed 80% of max_connections.

### 4. Long-Running Queries (> 5 minutes)
```sql
SELECT pid,
       now() - pg_stat_activity.query_start AS duration,
       query,
       state,
       wait_event_type,
       wait_event,
       usename,
       application_name,
       client_addr
FROM pg_stat_activity
WHERE (now() - pg_stat_activity.query_start) > interval '5 minutes'
  AND state <> 'idle'
ORDER BY duration DESC;
```

### 5. Blocking & Blocked Queries
```sql
SELECT blocked.pid          AS blocked_pid,
       blocked.usename       AS blocked_user,
       blocked.query         AS blocked_query,
       blocking.pid          AS blocking_pid,
       blocking.usename      AS blocking_user,
       blocking.query        AS blocking_query,
       blocked.wait_event_type,
       blocked.wait_event
FROM pg_stat_activity AS blocked
JOIN pg_stat_activity AS blocking
  ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE cardinality(pg_blocking_pids(blocked.pid)) > 0;
```

### 6. Lock Contention Detail
```sql
SELECT l.pid,
       l.locktype,
       l.relation::regclass,
       l.mode,
       l.granted,
       a.query,
       a.state,
       now() - a.query_start AS duration
FROM pg_locks l
JOIN pg_stat_activity a ON a.pid = l.pid
WHERE NOT l.granted
ORDER BY duration DESC;
```

### 7. Database Cache Hit Ratio
```sql
SELECT datname,
       blks_hit,
       blks_read,
       round(100.0 * blks_hit / NULLIF(blks_hit + blks_read, 0), 2) AS cache_hit_ratio,
       xact_commit,
       xact_rollback,
       deadlocks,
       temp_files,
       pg_size_pretty(temp_bytes) AS temp_size
FROM pg_stat_database
WHERE datname NOT IN ('template0', 'template1')
ORDER BY datname;
```
**Threshold:** Cache hit ratio should be > 99% for OLTP workloads.

### 8. Table Cache Hit Ratios
```sql
SELECT schemaname,
       relname,
       heap_blks_read,
       heap_blks_hit,
       round(100.0 * heap_blks_hit / NULLIF(heap_blks_hit + heap_blks_read, 0), 2) AS hit_pct,
       idx_blks_read,
       idx_blks_hit,
       round(100.0 * idx_blks_hit / NULLIF(idx_blks_hit + idx_blks_read, 0), 2) AS idx_hit_pct
FROM pg_statio_user_tables
ORDER BY heap_blks_read DESC
LIMIT 20;
```

### 9. Table Bloat Estimate
```sql
SELECT schemaname,
       tablename,
       pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename)) AS total_size,
       pg_size_pretty(pg_relation_size(schemaname || '.' || tablename)) AS table_size,
       n_dead_tup,
       n_live_tup,
       round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 1) AS dead_pct,
       last_vacuum,
       last_autovacuum,
       last_analyze,
       last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 20;
```
**Threshold:** dead_pct > 10% warrants manual VACUUM.

### 10. Index Usage — Unused Indexes
```sql
SELECT schemaname,
       tablename,
       indexname,
       idx_scan,
       idx_tup_read,
       idx_tup_fetch,
       pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND schemaname NOT IN ('pg_catalog', 'pg_toast')
ORDER BY pg_relation_size(indexrelid) DESC;
```

### 11. Table Sizes
```sql
SELECT schemaname,
       tablename,
       pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename)) AS total_size,
       pg_size_pretty(pg_relation_size(schemaname || '.' || tablename)) AS table_only,
       pg_size_pretty(pg_indexes_size(schemaname || '.' || tablename)) AS indexes_size,
       pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename)
                    - pg_relation_size(schemaname || '.' || tablename)
                    - pg_indexes_size(schemaname || '.' || tablename)) AS toast_size
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_total_relation_size(schemaname || '.' || tablename) DESC
LIMIT 20;
```

### 12. Checkpoint & BGWriter Stats
```sql
SELECT checkpoints_timed,
       checkpoints_req,
       checkpoint_write_time,
       checkpoint_sync_time,
       buffers_checkpoint,
       buffers_clean,
       buffers_backend,
       buffers_backend_fsync,
       buffers_alloc,
       stats_reset
FROM pg_stat_bgwriter;
```
**Alert:** `buffers_backend_fsync > 0` means the backend is writing WAL directly — checkpoint is overloaded. `checkpoints_req` should be much lower than `checkpoints_timed`.

### 13. WAL Generation Rate (PG14+)
```sql
SELECT wal_records,
       wal_fpi,
       wal_bytes,
       wal_buffers_full,
       wal_write,
       wal_sync,
       wal_write_time,
       wal_sync_time
FROM pg_stat_wal;
```

### 14. Vacuum Progress
```sql
SELECT p.pid,
       a.query,
       p.phase,
       p.heap_blks_total,
       p.heap_blks_scanned,
       p.heap_blks_vacuumed,
       round(100.0 * p.heap_blks_vacuumed / NULLIF(p.heap_blks_total, 0), 1) AS pct_done,
       p.index_vacuum_count,
       p.num_dead_tuples
FROM pg_stat_progress_vacuum p
JOIN pg_stat_activity a ON a.pid = p.pid;
```

### 15. Replication Lag
```sql
SELECT client_addr,
       application_name,
       state,
       sent_lsn,
       write_lsn,
       flush_lsn,
       replay_lsn,
       write_lag,
       flush_lag,
       replay_lag,
       sync_state
FROM pg_stat_replication;
```

### 16. Transaction ID Wraparound Risk
```sql
SELECT datname,
       age(datfrozenxid) AS xid_age,
       2147483647 - age(datfrozenxid) AS xids_remaining,
       round(100.0 * age(datfrozenxid) / 2147483647, 1) AS pct_toward_wraparound
FROM pg_database
ORDER BY age(datfrozenxid) DESC;
```
**Alert:** age > 1.5 billion (about 70%) requires immediate attention.

### 17. Table XID Age (for freeze candidates)
```sql
SELECT schemaname,
       relname,
       age(relfrozenxid) AS table_xid_age,
       pg_size_pretty(pg_total_relation_size(oid)) AS size
FROM pg_class
JOIN pg_namespace ON relnamespace = pg_namespace.oid
WHERE relkind = 'r'
  AND schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY age(relfrozenxid) DESC
LIMIT 20;
```

### 18. pg_stat_statements — Top Queries by Total Time
```sql
-- Requires pg_stat_statements extension
SELECT left(query, 100) AS query_snippet,
       calls,
       round(total_exec_time::numeric, 2) AS total_ms,
       round(mean_exec_time::numeric, 2) AS avg_ms,
       round(stddev_exec_time::numeric, 2) AS stddev_ms,
       rows,
       round(100.0 * shared_blks_hit /
             NULLIF(shared_blks_hit + shared_blks_read, 0), 2) AS hit_pct
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

---

## Key Metrics & Thresholds

| Metric | Warning | Critical |
|--------|---------|---------|
| Connection usage | > 70% of max_connections | > 85% |
| Cache hit ratio | < 99% | < 95% |
| Dead tuple % | > 5% | > 20% |
| XID age | > 1B | > 1.5B |
| Replication lag | > 10s | > 60s |
| Long queries | > 1 min | > 5 min |
| checkpoints_req > checkpoints_timed ratio | > 10% | > 30% |
| Temp file usage | Any | > 1GB/query |

---

## Best Practices

1. **Enable `pg_stat_statements`** in `shared_preload_libraries` on all production instances.
2. **Monitor connection pooling** — use PgBouncer; raw connections are expensive.
3. **Set `log_min_duration_statement`** (e.g., 1000ms) to capture slow queries in logs.
4. **Track `pg_stat_bgwriter.buffers_backend`** — rising values indicate `shared_buffers` is too small or checkpoint tuning is needed.
5. **Automate XID wraparound checks** — database forced shutdown at XID age ~2.1B.
6. **Reset stats periodically** with `pg_stat_reset()` after bulk operations to avoid skewed averages.
7. **Use `pg_stat_progress_*` views** to monitor long operations without `ps` access.
8. **Monitor `temp_files` and `temp_bytes`** in `pg_stat_database` — high values indicate `work_mem` is too low.

---

## Common Issues & Solutions

| Issue | Diagnostic Query | Solution |
|-------|-----------------|---------|
| Too many connections | `pg_stat_activity` count | Deploy PgBouncer, reduce max_connections |
| Slow queries | `pg_stat_statements` | EXPLAIN ANALYZE, add indexes |
| High I/O | `pg_statio_user_tables` | Increase shared_buffers, add indexes |
| Table bloat | `pg_stat_user_tables` n_dead_tup | Manual VACUUM, tune autovacuum |
| Lock waits | `pg_locks` + `pg_stat_activity` | Optimize transactions, reduce lock scope |
| XID wraparound | `age(datfrozenxid)` | Emergency VACUUM FREEZE |
| Checkpoint pressure | `pg_stat_bgwriter` | Increase checkpoint_completion_target, max_wal_size |

---

## Health Check Script Pattern

```sql
-- Quick health summary
SELECT
    (SELECT count(*) FROM pg_stat_activity WHERE state = 'active') AS active_queries,
    (SELECT count(*) FROM pg_stat_activity WHERE state = 'idle in transaction') AS idle_in_txn,
    (SELECT count(*) FROM pg_stat_activity WHERE wait_event_type = 'Lock') AS lock_waits,
    (SELECT round(100.0 * blks_hit / NULLIF(blks_hit + blks_read, 0), 2)
     FROM pg_stat_database WHERE datname = current_database()) AS cache_hit_pct,
    (SELECT age(datfrozenxid) FROM pg_database WHERE datname = current_database()) AS xid_age,
    (SELECT pg_size_pretty(pg_database_size(current_database()))) AS db_size,
    pg_postmaster_start_time() AS started_at,
    now() - pg_postmaster_start_time() AS uptime;
```

---

## References
- PostgreSQL Documentation: Monitoring Database Activity (https://www.postgresql.org/docs/current/monitoring-stats.html)
- `pg_stat_statements` module (https://www.postgresql.org/docs/current/pgstatstatements.html)
- PostgreSQL Monitoring Checklists (https://pganalyze.com/docs/checks)
