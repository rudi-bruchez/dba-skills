# PostgreSQL Diagnostic Query Library

Complete ready-to-run SQL queries for all health check areas.

---

## Server Basics

```sql
-- Version and uptime
SELECT version();
SELECT pg_postmaster_start_time(),
       now() - pg_postmaster_start_time() AS uptime;

-- Database sizes
SELECT datname,
       pg_size_pretty(pg_database_size(datname)) AS size
FROM pg_database
WHERE datname NOT IN ('template0', 'template1')
ORDER BY pg_database_size(datname) DESC;

-- Table sizes (top 20)
SELECT schemaname,
       tablename,
       pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename)) AS total_size,
       pg_size_pretty(pg_relation_size(schemaname || '.' || tablename)) AS table_only,
       pg_size_pretty(pg_indexes_size(schemaname || '.' || tablename)) AS indexes_size
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_total_relation_size(schemaname || '.' || tablename) DESC
LIMIT 20;
```

---

## Connections

```sql
-- Connection count vs max_connections
SELECT count(*) AS current_connections,
       (SELECT setting::int FROM pg_settings WHERE name = 'max_connections') AS max_connections,
       round(100.0 * count(*) /
           (SELECT setting::int FROM pg_settings WHERE name = 'max_connections'), 1) AS pct_used
FROM pg_stat_activity;

-- Connections by state and wait event
SELECT state, wait_event_type, wait_event, count(*)
FROM pg_stat_activity
WHERE pid <> pg_backend_pid()
GROUP BY state, wait_event_type, wait_event
ORDER BY count DESC;

-- Connections by application and user
SELECT usename, application_name, state, count(*)
FROM pg_stat_activity
GROUP BY usename, application_name, state
ORDER BY count DESC;

-- Long idle-in-transaction sessions (often blocking autovacuum)
SELECT pid, usename, application_name, client_addr,
       now() - state_change AS idle_in_txn_duration,
       left(query, 100) AS last_query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY idle_in_txn_duration DESC;
```

---

## Active Queries and Wait Events

```sql
-- Long-running active queries (over 5 minutes)
SELECT pid,
       now() - query_start AS duration,
       state,
       wait_event_type,
       wait_event,
       usename,
       application_name,
       client_addr,
       left(query, 150) AS query
FROM pg_stat_activity
WHERE now() - query_start > interval '5 minutes'
  AND state <> 'idle'
ORDER BY duration DESC;

-- All active queries with wait events
SELECT pid, usename, state, wait_event_type, wait_event,
       now() - query_start AS duration,
       left(query, 100) AS query
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY duration DESC NULLS LAST;

-- Wait event summary (what are connections waiting for?)
SELECT wait_event_type, wait_event, count(*) AS sessions
FROM pg_stat_activity
WHERE state = 'active'
  AND wait_event IS NOT NULL
GROUP BY wait_event_type, wait_event
ORDER BY sessions DESC;
```

---

## Lock Contention

```sql
-- Blocking and blocked queries
SELECT
    blocked.pid          AS blocked_pid,
    blocked.usename      AS blocked_user,
    left(blocked.query, 100)  AS blocked_query,
    blocking.pid         AS blocking_pid,
    blocking.usename     AS blocking_user,
    left(blocking.query, 100) AS blocking_query,
    now() - blocked.query_start AS blocked_for
FROM pg_stat_activity AS blocked
JOIN pg_stat_activity AS blocking
  ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE cardinality(pg_blocking_pids(blocked.pid)) > 0
ORDER BY blocked_for DESC;

-- Lock detail: which locks are not granted
SELECT l.pid, l.locktype, l.relation::regclass, l.mode, l.granted,
       a.usename, now() - a.query_start AS duration,
       left(a.query, 80) AS query
FROM pg_locks l
JOIN pg_stat_activity a ON a.pid = l.pid
WHERE NOT l.granted
ORDER BY duration DESC;

-- All locks on a specific table
SELECT l.pid, l.locktype, l.mode, l.granted,
       a.usename, a.state, left(a.query, 80) AS query
FROM pg_locks l
JOIN pg_stat_activity a ON a.pid = l.pid
WHERE l.relation = 'public.orders'::regclass
ORDER BY l.granted DESC;
```

---

## Cache Hit Ratio

```sql
-- Per-database cache hit ratio
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

-- Per-table cache hit ratio (tables with most I/O)
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

---

## Table Bloat and Autovacuum

```sql
-- Dead tuple ratio per table (top bloated tables)
SELECT schemaname,
       relname,
       n_dead_tup,
       n_live_tup,
       round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 1) AS dead_pct,
       pg_size_pretty(pg_total_relation_size(schemaname || '.' || relname)) AS total_size,
       last_vacuum,
       last_autovacuum,
       last_analyze,
       last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 20;

-- Tables where autovacuum hasn't run recently
SELECT schemaname, relname, n_dead_tup, last_autovacuum,
       now() - last_autovacuum AS since_autovacuum,
       autovacuum_count
FROM pg_stat_user_tables
WHERE (last_autovacuum IS NULL OR last_autovacuum < now() - interval '24 hours')
  AND n_dead_tup > 1000
ORDER BY n_dead_tup DESC;

-- Currently running autovacuums
SELECT p.pid,
       p.relid::regclass AS table_name,
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

---

## Index Health

```sql
-- Unused indexes (idx_scan = 0 since last stats reset)
SELECT schemaname,
       tablename,
       indexname,
       idx_scan,
       pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND schemaname NOT IN ('pg_catalog', 'pg_toast')
ORDER BY pg_relation_size(indexrelid) DESC;

-- Index usage statistics
SELECT schemaname,
       tablename,
       indexname,
       idx_scan,
       idx_tup_read,
       idx_tup_fetch,
       pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE schemaname NOT IN ('pg_catalog', 'pg_toast')
ORDER BY pg_relation_size(indexrelid) DESC
LIMIT 30;

-- Duplicate/redundant indexes (same table and key columns)
SELECT indrelid::regclass AS table_name,
       array_agg(indexrelid::regclass) AS indexes,
       array_agg(pg_get_indexdef(indexrelid)) AS definitions
FROM pg_index
GROUP BY indrelid, indkey
HAVING count(*) > 1;
```

---

## Transaction ID Wraparound

```sql
-- Database-level XID age
SELECT datname,
       age(datfrozenxid)                               AS xid_age,
       2147483647 - age(datfrozenxid)                  AS xids_remaining,
       round(100.0 * age(datfrozenxid) / 2147483647, 1) AS pct_toward_wraparound
FROM pg_database
ORDER BY age(datfrozenxid) DESC;

-- Table-level XID age (find tables needing freeze)
SELECT schemaname,
       relname,
       age(relfrozenxid)                                AS table_xid_age,
       pg_size_pretty(pg_total_relation_size(oid))      AS size
FROM pg_class
JOIN pg_namespace ON relnamespace = pg_namespace.oid
WHERE relkind = 'r'
  AND schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY age(relfrozenxid) DESC
LIMIT 20;
```

---

## Replication

```sql
-- Replication status on primary
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
       pg_size_pretty(pg_wal_lsn_diff(sent_lsn, replay_lsn)) AS replay_lag_size,
       sync_state
FROM pg_stat_replication;

-- Replication slot health
SELECT slot_name, slot_type, active,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_wal,
       wal_status
FROM pg_replication_slots;

-- Standby lag (run on standby)
SELECT pg_is_in_recovery()                                 AS is_standby,
       pg_last_wal_receive_lsn()                           AS received_lsn,
       pg_last_wal_replay_lsn()                            AS replayed_lsn,
       pg_wal_lsn_diff(pg_last_wal_receive_lsn(),
                       pg_last_wal_replay_lsn())           AS lag_bytes,
       now() - pg_last_xact_replay_timestamp()             AS replay_lag_time;
```

---

## WAL Archiving

```sql
-- Archive status
SELECT archived_count,
       last_archived_wal,
       last_archived_time,
       failed_count,
       last_failed_wal,
       last_failed_time,
       now() - last_archived_time AS time_since_last_archive
FROM pg_stat_archiver;
```

**Alert:** `failed_count > 0` or `time_since_last_archive > 10 minutes` on a busy database.

---

## BGWriter and Checkpoint

```sql
SELECT checkpoints_timed,
       checkpoints_req,
       round(100.0 * checkpoints_req /
           NULLIF(checkpoints_timed + checkpoints_req, 0), 1) AS req_checkpoint_pct,
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

---

## pg_stat_statements (requires extension)

```sql
-- Top queries by total execution time
SELECT left(query, 120) AS query_snippet,
       calls,
       round(total_exec_time::numeric / 1000, 2) AS total_sec,
       round(mean_exec_time::numeric, 2) AS avg_ms,
       round(stddev_exec_time::numeric, 2) AS stddev_ms,
       rows / NULLIF(calls, 0) AS rows_per_call,
       round(100.0 * shared_blks_hit /
           NULLIF(shared_blks_hit + shared_blks_read, 0), 2) AS cache_hit_pct
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;

-- Top queries by I/O (most disk reads)
SELECT left(query, 120) AS query_snippet,
       calls,
       shared_blks_read + local_blks_read AS total_reads,
       shared_blks_hit,
       round(100.0 * shared_blks_hit /
           NULLIF(shared_blks_hit + shared_blks_read, 0), 2) AS cache_hit_pct
FROM pg_stat_statements
ORDER BY (shared_blks_read + local_blks_read) DESC
LIMIT 10;
```
