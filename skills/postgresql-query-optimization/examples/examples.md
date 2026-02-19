# PostgreSQL Query Optimization Examples

## Scenario 1: Sequential Scan on a Large Table

**Situation:** A query that filters orders by customer_id takes 8 seconds. The orders table has 50 million rows.

**Step 1 — Check for sequential scans on large tables:**
```sql
SELECT relname, seq_scan, idx_scan,
       round(100.0 * seq_scan / NULLIF(seq_scan + idx_scan, 0), 1) AS seq_pct,
       pg_size_pretty(pg_total_relation_size('public.' || relname)) AS size
FROM pg_stat_user_tables
WHERE pg_total_relation_size('public.' || relname) > 100 * 1024 * 1024  -- > 100 MB
ORDER BY seq_scan DESC;
```
Result: `orders: seq_scan = 15432, idx_scan = 2, seq_pct = 99.9%, size = 8.2 GB`

**Step 2 — Capture the slow query plan:**
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT o.id, o.status, o.total
FROM orders o
WHERE o.customer_id = 42
ORDER BY o.created_at DESC
LIMIT 10;
```

Plan output:
```
Limit  (cost=180234.5..180234.6 rows=10 width=40)
      (actual time=8145.2..8145.3 rows=10 loops=1)
  ->  Sort  (cost=180234.5..180238.0 rows=1400 width=40)
            (actual time=8145.1..8145.2 rows=10 loops=1)
        Sort Key: created_at DESC
        Sort Method: top-N heapsort  Memory: 25kB
        ->  Seq Scan on orders  (cost=0.0..180180.0 rows=1400 width=40)
                               (actual time=0.02..8120.4 rows=1350 loops=1)
              Filter: (customer_id = 42)
              Rows Removed by Filter: 49998650
  Planning Time: 0.5 ms
  Execution Time: 8145.4 ms
```

**Reading the plan:** The Seq Scan reads all 50M rows and discards 49.9M. No index on `customer_id`.

**Step 3 — Check existing indexes:**
```sql
SELECT indexname, indexdef
FROM pg_indexes
WHERE tablename = 'orders';
```
Result: Only a primary key index on `id`. No index on `customer_id`.

**Step 4 — Create the index (non-blocking):**
```sql
CREATE INDEX CONCURRENTLY idx_orders_customer_date
    ON orders (customer_id, created_at DESC);
```

**Step 5 — Verify the new plan:**
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT o.id, o.status, o.total
FROM orders o
WHERE o.customer_id = 42
ORDER BY o.created_at DESC
LIMIT 10;
```

New plan:
```
Limit  (cost=0.56..9.81 rows=10 width=40)
      (actual time=0.04..0.07 rows=10 loops=1)
  ->  Index Scan using idx_orders_customer_date on orders
          (cost=0.56..1295.1 rows=1400 width=40)
          (actual time=0.03..0.06 rows=10 loops=1)
        Index Cond: (customer_id = 42)
  Planning Time: 0.6 ms
  Execution Time: 0.12 ms
```
Improvement: 8145ms down to 0.12ms — **67,000x faster**.

---

## Scenario 2: JSONB Query Optimization with GIN Index

**Situation:** An events table stores JSON payloads. A query filtering by event type is slow.

```sql
-- Slow query
SELECT id, payload, created_at
FROM events
WHERE payload->>'type' = 'user_login'
  AND created_at > now() - interval '24 hours'
ORDER BY created_at DESC;
-- Execution time: 45 seconds on 100M rows
```

**Step 1 — Run EXPLAIN:**
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, payload, created_at
FROM events
WHERE payload->>'type' = 'user_login'
  AND created_at > now() - interval '24 hours';
```
Shows `Seq Scan on events` with `Filter: ((payload->>'type') = 'user_login')`.

**Step 2 — Choose the right index approach:**

Option A — Expression index for exact key access (smaller, faster for this pattern):
```sql
CREATE INDEX CONCURRENTLY idx_events_type
    ON events ((payload->>'type'));
```

Option B — GIN index for containment queries (covers more operators):
```sql
CREATE INDEX CONCURRENTLY idx_events_gin
    ON events USING GIN (payload);
-- Query must use containment: WHERE payload @> '{"type": "user_login"}'
```

Also add a BRIN index for the time filter (events are inserted in time order):
```sql
CREATE INDEX CONCURRENTLY idx_events_created_brin
    ON events USING BRIN (created_at);
```

**Step 3 — Rewrite query to use containment (for GIN option):**
```sql
SELECT id, payload, created_at
FROM events
WHERE payload @> '{"type": "user_login"}'
  AND created_at > now() - interval '24 hours'
ORDER BY created_at DESC;
-- Execution time: 0.8 seconds
```

**Step 4 — Verify cache hit ratio for expression index option:**
```sql
-- Check the index is being used
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, payload, created_at
FROM events
WHERE payload->>'type' = 'user_login'
  AND created_at > now() - interval '24 hours';
```
Now shows `Index Scan using idx_events_type on events`.

---

## Scenario 3: Partial Index for Hot Rows

**Situation:** A jobs table has 10 million rows but 99% are in `completed` status. All application queries filter on `status = 'pending'` or `status = 'processing'`. Queries are slow and the index on `status` is not being used.

```sql
-- Full index on status: planner does not use it because too many rows match each status
CREATE INDEX idx_jobs_status ON jobs (status);
-- idx_scan = 12  (the planner rarely uses this index)
```

**Why this happens:**
```sql
SELECT status, count(*) FROM jobs GROUP BY status;
-- completed:   9,900,000  (99%)
-- pending:        50,000  (0.5%)
-- processing:     50,000  (0.5%)
```
The index on `status` has poor selectivity because most values are `completed`. The planner sees that an index scan on `status = 'pending'` would return 0.5% of rows but does not understand the distribution well.

**Step 1 — Drop the ineffective index:**
```sql
DROP INDEX CONCURRENTLY idx_jobs_status;
```

**Step 2 — Create a partial index covering only pending and processing jobs:**
```sql
CREATE INDEX CONCURRENTLY idx_jobs_active
    ON jobs (created_at, worker_id)
    WHERE status IN ('pending', 'processing');
```

This index contains only 100,000 rows (1% of the table). It is tiny and always in cache.

**Step 3 — Verify query uses the partial index:**
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, created_at, worker_id
FROM jobs
WHERE status = 'pending'
  AND created_at < now() - interval '5 minutes'
ORDER BY created_at;
```
Now shows `Index Scan using idx_jobs_active on jobs`. Execution time drops from 3 seconds to 2ms.

**Important:** Every query that should use this index must include `status IN ('pending', 'processing')` or `status = 'pending'` in its WHERE clause. The planner only uses partial indexes when it can prove the index condition is satisfied.

---

## Scenario 4: Statistics Fix for Correlated Columns

**Situation:** A query filtering on both `state` and `city` returns 50 rows but the planner estimates 50,000. This causes it to choose a Hash Join instead of a Nested Loop.

**Step 1 — Observe the misestimation:**
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT c.name, o.total
FROM customers c
JOIN orders o ON o.customer_id = c.id
WHERE c.state = 'NY' AND c.city = 'New York';
```
Output shows: `Rows: 50 (estimated: 50,000)`. The planner multiplied individual selectivities, not realizing state and city are correlated.

**Step 2 — Check column statistics:**
```sql
SELECT attname, n_distinct, correlation
FROM pg_stats
WHERE tablename = 'customers'
  AND attname IN ('state', 'city');
```

**Step 3 — Create extended statistics for the correlated pair:**
```sql
CREATE STATISTICS customers_location_stats (ndistinct, dependencies, mcv)
    ON state, city FROM customers;
ANALYZE customers;
```

**Step 4 — Verify improved estimates:**
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT c.name, o.total
FROM customers c
JOIN orders o ON o.customer_id = c.id
WHERE c.state = 'NY' AND c.city = 'New York';
```
Now shows estimated rows close to actual rows. The planner chooses Nested Loop instead of Hash Join, reducing execution time from 2 seconds to 80ms.

---

## Scenario 5: Analyzing pg_stat_statements to Find Problem Queries

**Situation:** The application team reports "the database is slow". No specific query is identified.

**Step 1 — Find the top time consumers:**
```sql
SELECT left(query, 120) AS query,
       calls,
       round(total_exec_time::numeric / 1000, 2) AS total_sec,
       round(mean_exec_time::numeric, 2) AS avg_ms,
       round(stddev_exec_time::numeric, 2) AS stddev_ms,
       rows / NULLIF(calls, 0) AS rows_per_call
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

Result shows:
- Query A: `total_sec = 12400`, `calls = 8000000`, `avg_ms = 1.55` — high frequency but fast
- Query B: `total_sec = 9800`, `calls = 200`, `avg_ms = 49000` — low frequency but very slow (49 seconds avg!)
- Query C: `total_sec = 6200`, `calls = 500000`, `avg_ms = 12.4`, `stddev_ms = 890` — inconsistent

**Step 2 — Focus on Query B (highest average time):**
The query text shows it is a report generating aggregates across 3 years of data.

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT date_trunc('month', order_date), sum(total), count(*)
FROM orders
WHERE order_date >= '2022-01-01'
GROUP BY 1
ORDER BY 1;
```

Shows `Seq Scan on orders` reading 150M rows with `Sort Method: external merge Disk: 256MB`. Two problems: no partition pruning and work_mem is too low.

**Fix 1 — Increase work_mem for the reporting role:**
```sql
ALTER ROLE reporter SET work_mem = '512MB';
```

**Fix 2 — Partition the orders table by month (future improvement for new data).**

**Step 3 — Investigate Query C (high standard deviation = inconsistent):**

High stddev with low average often means the query is fast when hitting cache but slow on cache miss. Or it runs fast usually but occasionally hits a lock.

```sql
SELECT left(query, 120) AS query, calls, avg_ms, stddev_ms,
       round(stddev_ms / NULLIF(avg_ms, 0), 2) AS coeff_of_variation
FROM pg_stat_statements
WHERE calls > 1000
ORDER BY coeff_of_variation DESC
LIMIT 10;
```

High `coeff_of_variation` (> 5) indicates lock contention or plan instability. Enable `auto_explain` to capture slow executions:
```ini
shared_preload_libraries = 'auto_explain'
auto_explain.log_min_duration = 5000   # Log queries > 5 seconds
auto_explain.log_analyze = on
auto_explain.log_buffers = on
```

**Step 4 — Reset statistics after fixes are in place:**
```sql
SELECT pg_stat_statements_reset();
-- Monitor for 24 hours, then re-check rankings
```
