# PostgreSQL Health Thresholds and Remediation

## Threshold Reference Table

| Metric | Query | Healthy | Warning | Critical | Remediation |
|--------|-------|---------|---------|---------|-------------|
| Connection usage | `count(*) / max_connections` | < 70% | 70–85% | > 85% | Deploy PgBouncer; audit idle connections |
| Cache hit ratio | `blks_hit / (blks_hit + blks_read)` | > 99% | 95–99% | < 95% | Increase `shared_buffers`; add indexes |
| Dead tuple pct | `n_dead_tup / (n_live_tup + n_dead_tup)` | < 5% | 5–20% | > 20% | Run `VACUUM ANALYZE`; tune autovacuum |
| XID age | `age(datfrozenxid)` | < 500M | > 1B | > 1.5B | Emergency `VACUUM FREEZE` |
| Replication lag (time) | `replay_lag` | < 10s | 10–60s | > 60s | Check standby I/O; increase `wal_keep_size` |
| Replication lag (bytes) | `pg_wal_lsn_diff(sent_lsn, replay_lsn)` | < 10MB | 10–100MB | > 100MB | Check network; check standby disk I/O |
| Long queries | `now() - query_start` | < 1 min | 1–5 min | > 5 min | Cancel or terminate; check for lock waits |
| Idle in transaction | `now() - state_change` (idle in txn) | < 30s | 30s–5 min | > 5 min | Kill the session; fix application transaction handling |
| Temp file size | `temp_bytes` per query | 0 | Any | > 1 GB | Increase `work_mem`; optimize query |
| checkpoints_req ratio | `checkpoints_req / total_checkpoints` | < 10% | 10–30% | > 30% | Increase `max_wal_size` |
| buffers_backend_fsync | `pg_stat_bgwriter.buffers_backend_fsync` | 0 | > 0 | > 10 | Increase `shared_buffers`; tune checkpoints |
| Replication slot WAL | `pg_wal_lsn_diff(current, restart_lsn)` | < 1GB | 1–10GB | > 10GB | Drop dead slots; fix lagging subscriber |
| Lock waits | Sessions with `wait_event_type = 'Lock'` | 0 | 1–5 | > 5 | Identify and terminate long blocking transactions |

---

## Connection Exhaustion

**Symptom:** New connections fail with `FATAL: remaining connection slots are reserved for non-replication superuser connections`.

**Diagnosis:**
```sql
SELECT count(*) AS current,
       (SELECT setting::int FROM pg_settings WHERE name = 'max_connections') AS max,
       count(*) FILTER (WHERE state = 'idle in transaction') AS idle_in_txn,
       count(*) FILTER (WHERE state = 'idle') AS idle
FROM pg_stat_activity;
```

**Remediation steps:**
1. Kill stale idle-in-transaction sessions immediately:
   ```sql
   SELECT pg_terminate_backend(pid)
   FROM pg_stat_activity
   WHERE state = 'idle in transaction'
     AND now() - state_change > interval '5 minutes';
   ```
2. Kill purely idle connections beyond a threshold:
   ```sql
   SELECT pg_terminate_backend(pid)
   FROM pg_stat_activity
   WHERE state = 'idle'
     AND now() - state_change > interval '30 minutes'
     AND usename <> 'postgres';
   ```
3. Deploy PgBouncer in transaction pooling mode to multiplex connections.
4. Set `idle_in_transaction_session_timeout = 300000` (5 minutes in ms) in `postgresql.conf` to prevent recurrence.

---

## High Dead Tuple Percentage

**Symptom:** Queries are slow; table scans read many dead tuples; disk usage grows without data growth.

**Diagnosis:**
```sql
SELECT relname, n_dead_tup, n_live_tup,
       round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 1) AS dead_pct,
       last_autovacuum
FROM pg_stat_user_tables
WHERE round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 1) > 5
ORDER BY dead_pct DESC;
```

**Remediation steps:**
1. Run manual vacuum immediately for tables above 20%:
   ```sql
   VACUUM VERBOSE schema.tablename;
   ```
2. Check if autovacuum is being blocked by a long transaction:
   ```sql
   SELECT pid, now() - xact_start AS txn_age, state, left(query, 80) AS query
   FROM pg_stat_activity
   WHERE xact_start IS NOT NULL
   ORDER BY xact_start;
   ```
3. If autovacuum scale factor is too large for a busy table, reduce it:
   ```sql
   ALTER TABLE orders SET (
       autovacuum_vacuum_scale_factor = 0.02,
       autovacuum_analyze_scale_factor = 0.01
   );
   ```
4. For severe bloat requiring space reclaim (locks the table):
   ```sql
   VACUUM FULL schema.tablename;
   -- Online alternative using pg_repack extension:
   -- pg_repack -t schema.tablename dbname
   ```

---

## Transaction ID Wraparound Risk

**Symptom:** PostgreSQL logs warnings: `WARNING: database "mydb" must be vacuumed within N transactions`. At XID age ~2.1 billion, PostgreSQL shuts down.

**Diagnosis:**
```sql
SELECT datname, age(datfrozenxid) AS xid_age,
       round(100.0 * age(datfrozenxid) / 2147483647, 1) AS pct
FROM pg_database ORDER BY age(datfrozenxid) DESC;
```

**Remediation by severity:**

XID age 500M–1B (normal): Verify autovacuum is running regularly.

XID age 1B–1.5B (warning):
```sql
-- Tune autovacuum_freeze_max_age globally
-- In postgresql.conf:
-- autovacuum_freeze_max_age = 150000000  (150M, forces earlier freeze)
-- Restart required for this setting
```

XID age above 1.5B (CRITICAL — immediate action):
```sql
-- Step 1: Identify the oldest tables
SELECT schemaname, relname, age(relfrozenxid) AS table_age
FROM pg_class JOIN pg_namespace ON relnamespace = pg_namespace.oid
WHERE relkind = 'r'
ORDER BY age(relfrozenxid) DESC LIMIT 10;

-- Step 2: Force freeze on oldest tables (or whole database)
-- This will take time; DO NOT stop it once started
VACUUM FREEZE VERBOSE schema.tablename;

-- Or freeze the entire database (connect as superuser):
VACUUM FREEZE;
```

---

## Low Cache Hit Ratio

**Symptom:** `blks_read` is high relative to `blks_hit`; I/O wait is high in the OS.

**Diagnosis:**
```sql
SELECT datname,
       round(100.0 * blks_hit / NULLIF(blks_hit + blks_read, 0), 2) AS cache_hit_pct
FROM pg_stat_database
WHERE datname = current_database();
```

**Remediation:**
1. Check `shared_buffers` setting — should be 25% of RAM:
   ```sql
   SHOW shared_buffers;
   -- Increase in postgresql.conf, requires restart:
   -- shared_buffers = 8GB  (on a 32GB system)
   ```
2. Check for missing indexes causing full table scans:
   ```sql
   SELECT relname, seq_scan, seq_tup_read,
          pg_size_pretty(pg_total_relation_size(schemaname || '.' || relname)) AS size
   FROM pg_stat_user_tables
   ORDER BY seq_scan DESC LIMIT 10;
   ```
3. Check `effective_cache_size` is set correctly (planner hint, does not allocate memory):
   ```sql
   SHOW effective_cache_size;
   -- Should be 50-75% of total RAM
   -- ALTER SYSTEM SET effective_cache_size = '24GB';
   ```

---

## High Replication Lag

**Symptom:** Standby lag growing; reads from standby return stale data.

**Diagnosis:**
```sql
-- On primary:
SELECT application_name, replay_lag,
       pg_size_pretty(pg_wal_lsn_diff(sent_lsn, replay_lsn)) AS lag_size,
       sync_state
FROM pg_stat_replication;
```

**Remediation by cause:**

Standby I/O bottleneck:
- Check disk I/O on standby with OS tools (`iostat`, `iotop`)
- Consider adding `max_parallel_maintenance_workers` on standby

Network latency:
- Check network bandwidth and latency between primary and standby
- Consider `wal_compression = on` on primary to reduce WAL volume

Long queries on standby cancelling replay:
```sql
-- On standby: set to avoid query cancellation due to replay conflicts
-- In postgresql.conf:
-- hot_standby_feedback = on        (tells primary what XIDs standby needs)
-- max_standby_streaming_delay = 60s (allow 60s delay before cancelling)
```

Replication slot not advancing:
```sql
SELECT slot_name, active, restart_lsn,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained
FROM pg_replication_slots
WHERE NOT active;
-- Drop inactive slots that belong to decommissioned standbys:
-- SELECT pg_drop_replication_slot('slot_name');
```

---

## Checkpoint Pressure

**Symptom:** `buffers_backend > 0`; high I/O during checkpoints; `checkpoints_req` is high.

**Diagnosis:**
```sql
SELECT checkpoints_timed, checkpoints_req,
       round(100.0 * checkpoints_req /
           NULLIF(checkpoints_timed + checkpoints_req, 0), 1) AS req_pct,
       buffers_backend, buffers_backend_fsync
FROM pg_stat_bgwriter;
```

**Remediation:**
```ini
# In postgresql.conf:
max_wal_size = 4GB                   # Increase to reduce checkpoints_req
checkpoint_completion_target = 0.9   # Spread I/O over 90% of interval
checkpoint_timeout = 10min           # Allow more time between checkpoints
```
