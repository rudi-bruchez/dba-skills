---
name: postgresql-health-check
description: Diagnoses PostgreSQL instance health by analyzing connections, wait events, lock contention, table and index bloat, autovacuum status, replication lag, and transaction ID wraparound risk. Use when PostgreSQL is slow, connections are being refused, tables are bloated, autovacuum is not keeping up, or during routine health audits.
metadata:
  author: dba-skills
  version: "1.0"
  platform: PostgreSQL 13+
---

# PostgreSQL Health Check

## Quick Health Check Workflow

Run these steps in order to triage a PostgreSQL instance:

1. **Server state** — check version, uptime, and database sizes
2. **Connections** — check usage vs `max_connections`
3. **Activity** — find long-running queries and idle-in-transaction sessions
4. **Locks** — identify blocking and blocked queries
5. **Cache hit ratio** — verify buffer cache effectiveness
6. **Bloat** — check dead tuple percentages
7. **Autovacuum** — confirm autovacuum is keeping up
8. **Wraparound** — check transaction ID age (critical for PostgreSQL)
9. **Replication** — check lag if standbys exist
10. **BGWriter/checkpoints** — check I/O health

### Step 1 — One-Line Health Summary

```sql
SELECT
    (SELECT count(*) FROM pg_stat_activity WHERE state = 'active') AS active_queries,
    (SELECT count(*) FROM pg_stat_activity WHERE state = 'idle in transaction') AS idle_in_txn,
    (SELECT count(*) FROM pg_stat_activity WHERE wait_event_type = 'Lock') AS lock_waits,
    (SELECT round(100.0 * blks_hit / NULLIF(blks_hit + blks_read, 0), 2)
     FROM pg_stat_database WHERE datname = current_database()) AS cache_hit_pct,
    (SELECT age(datfrozenxid) FROM pg_database WHERE datname = current_database()) AS xid_age,
    (SELECT pg_size_pretty(pg_database_size(current_database()))) AS db_size,
    now() - pg_postmaster_start_time() AS uptime;
```

---

## Connection and Activity Analysis

### Connection Usage vs Limit

```sql
SELECT
    count(*) AS current_connections,
    (SELECT setting::int FROM pg_settings WHERE name = 'max_connections') AS max_connections,
    round(100.0 * count(*) /
        (SELECT setting::int FROM pg_settings WHERE name = 'max_connections'), 1) AS pct_used
FROM pg_stat_activity;
```

**Threshold:** Alert when connections exceed 80% of `max_connections`. Reserve 3 superuser connections (`superuser_reserved_connections`).

### Connection Breakdown by State

```sql
SELECT state, wait_event_type, wait_event, count(*)
FROM pg_stat_activity
WHERE pid <> pg_backend_pid()
GROUP BY state, wait_event_type, wait_event
ORDER BY count DESC;
```

**Action:** `idle in transaction` sessions holding locks are often a source of blocking. Kill stale ones:
```sql
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND now() - state_change > interval '10 minutes';
```

### Long-Running Queries (over 5 minutes)

```sql
SELECT pid,
       now() - query_start AS duration,
       state,
       wait_event_type,
       wait_event,
       usename,
       left(query, 100) AS query_snippet
FROM pg_stat_activity
WHERE now() - query_start > interval '5 minutes'
  AND state <> 'idle'
ORDER BY duration DESC;
```

---

## Lock Contention and Blocking

### Find Blocking and Blocked Queries

```sql
SELECT
    blocked.pid          AS blocked_pid,
    blocked.usename      AS blocked_user,
    left(blocked.query, 80)  AS blocked_query,
    blocking.pid         AS blocking_pid,
    blocking.usename     AS blocking_user,
    left(blocking.query, 80) AS blocking_query,
    now() - blocked.query_start AS blocked_for
FROM pg_stat_activity AS blocked
JOIN pg_stat_activity AS blocking
  ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE cardinality(pg_blocking_pids(blocked.pid)) > 0
ORDER BY blocked_for DESC;
```

**Action:** If the blocking query is stuck, terminate it:
```sql
SELECT pg_terminate_backend(<blocking_pid>);
```

### Lock Detail

```sql
SELECT l.pid, l.locktype, l.relation::regclass, l.mode, l.granted,
       now() - a.query_start AS duration
FROM pg_locks l
JOIN pg_stat_activity a ON a.pid = l.pid
WHERE NOT l.granted
ORDER BY duration DESC;
```

---

## Cache Hit Ratio

```sql
SELECT datname,
       round(100.0 * blks_hit / NULLIF(blks_hit + blks_read, 0), 2) AS cache_hit_pct,
       deadlocks,
       temp_files,
       pg_size_pretty(temp_bytes) AS temp_size
FROM pg_stat_database
WHERE datname NOT IN ('template0', 'template1')
ORDER BY datname;
```

**Threshold:** Cache hit ratio must be above 99% for OLTP workloads. Below 95% is critical. High `temp_bytes` indicates `work_mem` is too low.

---

## Table and Index Bloat Assessment

### Dead Tuple Ratio (Fast Bloat Estimate)

```sql
SELECT schemaname,
       relname,
       n_dead_tup,
       n_live_tup,
       round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 1) AS dead_pct,
       pg_size_pretty(pg_total_relation_size(schemaname || '.' || relname)) AS total_size,
       last_autovacuum,
       last_vacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 20;
```

**Threshold:** `dead_pct` above 10% warrants investigation. Above 20% requires immediate `VACUUM`. If a table has not been vacuumed recently and dead_pct is rising, autovacuum may be blocked.

**Action — Run manual vacuum:**
```sql
VACUUM ANALYZE schema.tablename;
-- For severe bloat requiring full rewrite (locks table):
VACUUM FULL schema.tablename;
```

### Unused Indexes (Wasted Write Overhead)

```sql
SELECT schemaname, tablename, indexname, idx_scan,
       pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND schemaname NOT IN ('pg_catalog', 'pg_toast')
ORDER BY pg_relation_size(indexrelid) DESC;
```

---

## Autovacuum Status

### Tables Where Autovacuum Is Falling Behind

```sql
SELECT schemaname,
       relname,
       n_dead_tup,
       round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 1) AS dead_pct,
       last_autovacuum,
       now() - last_autovacuum AS since_last_autovacuum,
       autovacuum_count
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY n_dead_tup DESC
LIMIT 20;
```

### Currently Running Vacuums

```sql
SELECT p.pid,
       a.query,
       p.phase,
       round(100.0 * p.heap_blks_vacuumed / NULLIF(p.heap_blks_total, 0), 1) AS pct_done,
       p.num_dead_tuples
FROM pg_stat_progress_vacuum p
JOIN pg_stat_activity a ON a.pid = p.pid;
```

**Common cause of autovacuum blocking:** A long-running transaction prevents autovacuum from cleaning up dead tuples. Find it with the long-running queries query above.

**Per-table autovacuum tuning for high-churn tables:**
```sql
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.02,
    autovacuum_analyze_scale_factor = 0.01,
    autovacuum_vacuum_cost_delay = 2
);
```

---

## Replication Lag Check

Run this on the primary server:

```sql
SELECT client_addr,
       application_name,
       state,
       pg_size_pretty(pg_wal_lsn_diff(sent_lsn, replay_lsn)) AS replay_lag_size,
       replay_lag,
       flush_lag,
       write_lag,
       sync_state
FROM pg_stat_replication
ORDER BY pg_wal_lsn_diff(sent_lsn, replay_lsn) DESC;
```

Run this on a standby to check its own lag:

```sql
SELECT pg_is_in_recovery() AS is_standby,
       now() - pg_last_xact_replay_timestamp() AS replay_lag_time,
       pg_wal_lsn_diff(pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn()) AS lag_bytes;
```

**Threshold:** Replication lag above 10 seconds is a warning; above 60 seconds is critical.

---

## Transaction ID Wraparound Risk

This is one of PostgreSQL's most critical failure modes. At approximately 2.1 billion transaction IDs, PostgreSQL will forcibly shut down to prevent data corruption.

```sql
SELECT datname,
       age(datfrozenxid)                          AS xid_age,
       2147483647 - age(datfrozenxid)             AS xids_remaining,
       round(100.0 * age(datfrozenxid) / 2147483647, 1) AS pct_toward_wraparound
FROM pg_database
ORDER BY age(datfrozenxid) DESC;
```

**Thresholds:**
- Above 500 million: healthy, monitor
- Above 1 billion: warning, tune autovacuum
- Above 1.5 billion: CRITICAL — run emergency vacuum immediately

**Emergency action when XID age is critical:**
```sql
-- Run as superuser, this forces a freeze on the entire database
VACUUM FREEZE VERBOSE;
-- Or target specific tables with the oldest XID age:
SELECT schemaname, relname, age(relfrozenxid) AS table_xid_age
FROM pg_class
JOIN pg_namespace ON relnamespace = pg_namespace.oid
WHERE relkind = 'r'
ORDER BY age(relfrozenxid) DESC
LIMIT 10;
```

---

## BGWriter and Checkpoint Health

```sql
SELECT checkpoints_timed,
       checkpoints_req,
       round(100.0 * checkpoints_req /
           NULLIF(checkpoints_timed + checkpoints_req, 0), 1) AS req_pct,
       buffers_backend,
       buffers_backend_fsync,
       buffers_checkpoint,
       buffers_clean
FROM pg_stat_bgwriter;
```

**Alerts:**
- `buffers_backend_fsync > 0` — backends writing WAL directly; checkpoint is overloaded
- `checkpoints_req` more than 10% of total — checkpoint interval is too short; increase `max_wal_size`

---

## Health Metric Thresholds Summary

| Metric | Warning | Critical |
|--------|---------|---------|
| Connection usage | > 70% of max_connections | > 85% |
| Cache hit ratio | < 99% | < 95% |
| Dead tuple pct | > 10% | > 20% |
| XID age | > 1 billion | > 1.5 billion |
| Replication lag (time) | > 10 seconds | > 60 seconds |
| Long-running query | > 1 minute | > 5 minutes |
| Temp file usage | Any | > 1 GB per query |
| checkpoints_req ratio | > 10% of total | > 30% |

---

## Reference Files

- [Diagnostic query library](references/diagnostic-queries.md) — complete ready-to-run SQL for all health areas
- [pg_stat_* view reference](references/pg-stat-reference.md) — key views with column explanations
- [Thresholds and remediation](references/thresholds.md) — metric thresholds with remediation steps

## Examples

- [Health check scenarios](examples/examples.md) — connection exhaustion, bloat/wraparound, autovacuum lag, replication lag
