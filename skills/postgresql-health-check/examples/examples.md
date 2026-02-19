# PostgreSQL Health Check Examples

## Scenario 1: Connection Exhaustion

**Situation:** Application team reports "connection refused" errors. New connections to the PostgreSQL database are failing. The application is running normally on its side.

**Step 1 — Check connection usage:**
```sql
SELECT count(*) AS current,
       (SELECT setting::int FROM pg_settings WHERE name = 'max_connections') AS max_conn,
       round(100.0 * count(*) /
           (SELECT setting::int FROM pg_settings WHERE name = 'max_connections'), 1) AS pct_used
FROM pg_stat_activity;
```
Result: `current = 198, max_conn = 200, pct_used = 99.0` — connection pool is full.

**Step 2 — Find the culprit sessions:**
```sql
SELECT usename, application_name, state, count(*),
       max(now() - state_change) AS oldest_in_state
FROM pg_stat_activity
GROUP BY usename, application_name, state
ORDER BY count DESC;
```
Result: 85 sessions from `app_service` in state `idle in transaction`, many over 20 minutes old.

**Step 3 — Kill the stale idle-in-transaction sessions:**
```sql
SELECT pg_terminate_backend(pid), usename, now() - state_change AS age
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND now() - state_change > interval '5 minutes'
  AND usename = 'app_service';
```

**Step 4 — Prevent recurrence by setting a timeout:**
```sql
ALTER SYSTEM SET idle_in_transaction_session_timeout = '300000'; -- 5 minutes in ms
SELECT pg_reload_conf();
```

**Step 5 — Long-term fix:** Deploy PgBouncer in transaction pooling mode:
```ini
[databases]
mydb = host=localhost port=5432 dbname=mydb

[pgbouncer]
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 20
```
This allows 1000 application connections to share 20 actual PostgreSQL connections.

---

## Scenario 2: Table Bloat and Wraparound Risk

**Situation:** A nightly monitoring alert fires showing XID age is 1.2 billion on the production database. Some tables have over 30% dead tuples. Performance has been degrading for two weeks.

**Step 1 — Confirm wraparound risk:**
```sql
SELECT datname,
       age(datfrozenxid) AS xid_age,
       round(100.0 * age(datfrozenxid) / 2147483647, 1) AS pct_toward_wraparound
FROM pg_database
ORDER BY age(datfrozenxid) DESC;
```
Result: `xid_age = 1,241,000,000` (58% toward wraparound). Warning threshold crossed.

**Step 2 — Find the oldest tables:**
```sql
SELECT schemaname, relname,
       age(relfrozenxid) AS table_xid_age,
       pg_size_pretty(pg_total_relation_size(oid)) AS size
FROM pg_class
JOIN pg_namespace ON relnamespace = pg_namespace.oid
WHERE relkind = 'r'
  AND schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY age(relfrozenxid) DESC
LIMIT 10;
```
Result: `orders` table has age 1,240,000,000 and is 45 GB.

**Step 3 — Check if autovacuum is running on this table:**
```sql
SELECT relname, last_autovacuum, autovacuum_count, n_dead_tup,
       round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 1) AS dead_pct
FROM pg_stat_user_tables
WHERE relname = 'orders';
```
Result: `last_autovacuum = 3 days ago`, `dead_pct = 32%`.

**Step 4 — Check if a long transaction is blocking autovacuum:**
```sql
SELECT pid, usename, now() - xact_start AS txn_age, state, left(query, 80) AS query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start
LIMIT 5;
```
Result: A report job started 3 days ago is holding an open transaction.

**Step 5 — Terminate the blocking transaction and run vacuum:**
```sql
SELECT pg_terminate_backend(<pid_of_report_job>);
```

Then run vacuum immediately:
```bash
# From the OS, run in the background to not block the terminal
psql -d production -c "VACUUM FREEZE VERBOSE public.orders" &
```

**Step 6 — Tune autovacuum for this table:**
```sql
ALTER TABLE public.orders SET (
    autovacuum_vacuum_scale_factor = 0.02,   -- Trigger at 2% dead tuples (not 20%)
    autovacuum_vacuum_cost_delay = 2,
    autovacuum_vacuum_cost_limit = 800
);
```

---

## Scenario 3: Autovacuum Not Keeping Up

**Situation:** Dead tuple counts are growing on the `events` table despite autovacuum running. The table receives 5 million inserts and 2 million deletes per day.

**Step 1 — Observe the problem:**
```sql
SELECT relname, n_dead_tup, n_live_tup,
       round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 1) AS dead_pct,
       last_autovacuum, autovacuum_count,
       round(n_dead_tup::numeric / NULLIF(autovacuum_count, 0), 0) AS avg_dead_per_vacuum
FROM pg_stat_user_tables
WHERE relname = 'events';
```
Result: `n_dead_tup = 8,200,000`, `dead_pct = 18%`, `avg_dead_per_vacuum = 2,100,000` — autovacuum runs but never catches up.

**Step 2 — Check current autovacuum settings for this table:**
```sql
SELECT reloptions
FROM pg_class
WHERE relname = 'events';
```
Result: `NULL` — using global defaults (scale_factor = 0.20, cost_delay = 2ms).

**Step 3 — Calculate why autovacuum triggers too late:**

With 40M live rows and `scale_factor = 0.20`:
- Autovacuum trigger threshold = 50 + 0.20 * 40,000,000 = **8,000,050 dead tuples**
- By the time vacuum runs, there are already 8M dead tuples to clean

**Step 4 — Reduce the scale factor significantly:**
```sql
ALTER TABLE events SET (
    autovacuum_vacuum_scale_factor = 0.01,     -- Trigger at 1% (400,000 rows)
    autovacuum_analyze_scale_factor = 0.005,   -- Analyze at 0.5%
    autovacuum_vacuum_cost_delay = 0,           -- No I/O throttle (table is critical)
    autovacuum_vacuum_cost_limit = 1000
);
```

**Step 5 — Verify improvement after 24 hours:**
```sql
SELECT relname, n_dead_tup, last_autovacuum, autovacuum_count
FROM pg_stat_user_tables
WHERE relname = 'events';
```

---

## Scenario 4: Replication Lag Alert

**Situation:** Monitoring alerts that the read replica used for reporting queries is lagging 3 minutes behind the primary. Reports are returning stale data.

**Step 1 — Check lag on the primary:**
```sql
SELECT application_name,
       client_addr,
       state,
       replay_lag,
       pg_size_pretty(pg_wal_lsn_diff(sent_lsn, replay_lsn)) AS lag_bytes,
       sync_state
FROM pg_stat_replication;
```
Result: `replay_lag = 00:03:12`, `lag_bytes = 2.1 GB`.

**Step 2 — Check if WAL is being sent but not replayed:**

On the standby:
```sql
SELECT pg_last_wal_receive_lsn() AS received,
       pg_last_wal_replay_lsn()  AS replayed,
       pg_wal_lsn_diff(pg_last_wal_receive_lsn(),
                       pg_last_wal_replay_lsn())     AS pending_replay_bytes,
       now() - pg_last_xact_replay_timestamp()       AS replay_lag_time;
```
Result: `pending_replay_bytes = 2.1 GB` — WAL is received but replay is falling behind.

**Step 3 — Check standby for long-running queries blocking replay:**

On the standby:
```sql
SELECT pid, now() - query_start AS duration, state, left(query, 100) AS query
FROM pg_stat_activity
WHERE state <> 'idle'
ORDER BY duration DESC;
```
Result: A large reporting query has been running for 8 minutes. PostgreSQL is delaying WAL replay to avoid conflicting with this query.

**Step 4 — Options to resolve:**

Option A — Enable `hot_standby_feedback` to avoid replay conflicts (primary must retain rows longer):
```sql
-- On standby (in postgresql.conf):
-- hot_standby_feedback = on
-- Reload config:
SELECT pg_reload_conf();
```

Option B — Increase max standby delay before query cancellation:
```sql
-- On standby (in postgresql.conf):
-- max_standby_streaming_delay = 300s    (allow 5 minutes delay)
SELECT pg_reload_conf();
```

Option C — Cancel the blocking query on the standby (it will be re-run):
```sql
SELECT pg_cancel_backend(<pid>);
```

**Step 5 — Verify lag is recovering:**
```sql
-- On primary, watch lag reduce:
SELECT application_name, replay_lag,
       pg_size_pretty(pg_wal_lsn_diff(sent_lsn, replay_lsn)) AS lag_bytes
FROM pg_stat_replication;
```
