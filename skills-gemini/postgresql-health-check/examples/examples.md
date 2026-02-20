# PostgreSQL Health Check Examples

Practical scenarios and how to interpret diagnostic results.

## Scenario 1: Connection Saturation

### Diagnosis
1.  **Check Active Sessions:** Run `pg_stat_activity`.
2.  **Observation:** Found 95 out of 100 `max_connections` are in use.
3.  **State Analysis:** Many sessions are in `idle in transaction` state for a long time.
```sql
SELECT pid, usename, state, query_start, now() - query_start AS duration, query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY duration DESC;
```
4.  **Root Cause:** The application is opening transactions but not closing them correctly (e.g., missing `COMMIT` or `ROLLBACK`).
5.  **Solution:** Fix the application logic or set `idle_in_transaction_session_timeout` in `postgresql.conf` to automatically close them.

## Scenario 2: High Table Bloat

### Diagnosis
1.  **Check Bloat Status:** Monitor `pg_stat_user_tables`.
2.  **Observation:** The `Orders` table has 10 million live tuples and 5 million dead tuples.
```sql
SELECT relname, n_live_tup, n_dead_tup, last_vacuum, last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 0
ORDER BY n_dead_tup DESC;
```
3.  **Action:** Check if autovacuum has run recently.
4.  **Observation:** `last_autovacuum` was several days ago.
5.  **Root Cause:** A long-running transaction (e.g., a backup or a reporting query) is preventing autovacuum from cleaning up.
6.  **Solution:** Terminate the long-running transaction or adjust autovacuum settings to be more aggressive for this table.

## Scenario 3: High I/O Wait Events

### Diagnosis
1.  **Check Activity:** Monitor `wait_event_type` and `wait_event` in `pg_stat_activity`.
2.  **Observation:** Many queries have `wait_event_type = 'IO'` and `wait_event = 'DataFileRead'`.
3.  **Action:** Check the Cache Hit Ratio.
```sql
SELECT
    datname,
    100 * blks_hit / (blks_hit + blks_read) AS CacheHitRatio
FROM pg_stat_database;
```
4.  **Observation:** Cache Hit Ratio is only 75%.
5.  **Solution:** Increase `shared_buffers` if there is available RAM, or optimize the queries to reduce the amount of data being read from disk (e.g., add indexes).

## Scenario 4: Missing or Unused Indexes

### Diagnosis
1.  **Check Index Stats:** Monitor `pg_stat_user_indexes`.
2.  **Observation:** An index `idx_old_column` has `idx_scan = 0`.
```sql
SELECT relname, indexrelname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0 AND schemaname = 'public'
ORDER BY relname;
```
3.  **Action:** Verify if the index is really not needed.
4.  **Solution:** Drop the unused index to improve write performance and reduce storage and maintenance overhead.
5.  **Observation:** A large table `Transactions` has many `idx_scan` but also many sequential scans.
6.  **Solution:** Use `EXPLAIN ANALYZE` on queries against `Transactions` to identify where new indexes might be needed.
