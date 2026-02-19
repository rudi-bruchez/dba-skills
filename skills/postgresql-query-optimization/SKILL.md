---
name: postgresql-query-optimization
description: Analyzes and optimizes slow PostgreSQL queries using EXPLAIN ANALYZE, pg_stat_statements, index recommendations, and planner configuration. Use when queries are slow, pg_stat_statements shows expensive queries, sequential scans occur on large tables, or after schema changes cause plan regressions.
metadata:
  author: dba-skills
  version: "1.0"
  platform: PostgreSQL 13+
---

# PostgreSQL Query Optimization

## Workflow: Find and Fix Slow Queries

1. **Identify** — find expensive queries using `pg_stat_statements` or `pg_stat_activity`
2. **Capture the plan** — run `EXPLAIN (ANALYZE, BUFFERS)` on the slow query
3. **Read the plan** — identify the expensive nodes and misestimations
4. **Diagnose** — determine root cause: missing index, stale statistics, wrong config
5. **Fix** — create an index, run ANALYZE, or adjust configuration
6. **Verify** — re-run EXPLAIN and compare actual vs previous execution time

---

## Step 1 — Find Slow Queries

### Using pg_stat_statements (requires extension)

Enable in `postgresql.conf` and restart: `shared_preload_libraries = 'pg_stat_statements'`

```sql
-- Top queries by total execution time (biggest overall cost)
SELECT left(query, 120) AS query,
       calls,
       round(total_exec_time::numeric / 1000, 2) AS total_sec,
       round(mean_exec_time::numeric, 2)          AS avg_ms,
       round(stddev_exec_time::numeric, 2)        AS stddev_ms,
       rows / NULLIF(calls, 0)                    AS rows_per_call
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Top queries by average execution time (slowest individual runs)
SELECT left(query, 120) AS query,
       calls,
       round(mean_exec_time::numeric, 2) AS avg_ms,
       round(stddev_exec_time::numeric, 2) AS stddev_ms
FROM pg_stat_statements
WHERE calls > 10
ORDER BY mean_exec_time DESC
LIMIT 10;

-- Top queries by I/O (most disk reads)
SELECT left(query, 120) AS query,
       calls,
       shared_blks_read,
       round(100.0 * shared_blks_hit /
           NULLIF(shared_blks_hit + shared_blks_read, 0), 1) AS cache_hit_pct
FROM pg_stat_statements
ORDER BY shared_blks_read DESC
LIMIT 10;
```

### Find Currently Running Slow Queries

```sql
SELECT pid, now() - query_start AS duration,
       state, wait_event_type, wait_event,
       left(query, 150) AS query
FROM pg_stat_activity
WHERE state = 'active'
  AND now() - query_start > interval '5 seconds'
ORDER BY duration DESC;
```

### Find Tables with Excessive Sequential Scans

```sql
SELECT relname,
       seq_scan,
       seq_tup_read,
       idx_scan,
       round(100.0 * seq_scan / NULLIF(seq_scan + idx_scan, 0), 1) AS seq_scan_pct,
       pg_size_pretty(pg_total_relation_size(schemaname || '.' || relname)) AS size
FROM pg_stat_user_tables
WHERE seq_scan > 0
  AND pg_total_relation_size(schemaname || '.' || relname) > 10 * 1024 * 1024  -- > 10 MB
ORDER BY seq_scan DESC
LIMIT 20;
```

---

## Step 2 — Capture the Execution Plan

Always use `ANALYZE` and `BUFFERS` together — `EXPLAIN` alone shows only estimates.

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;
```

For complex or production queries, use `TIMING OFF` to reduce overhead:
```sql
EXPLAIN (ANALYZE, BUFFERS, TIMING OFF) SELECT ...;
```

For machine-readable output (useful for automated tools):
```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) SELECT ...;
```

**Important:** `EXPLAIN ANALYZE` actually executes the query. For DML statements, wrap in a transaction and roll back:
```sql
BEGIN;
EXPLAIN (ANALYZE, BUFFERS) UPDATE orders SET status = 'processed' WHERE id = 42;
ROLLBACK;
```

---

## Step 3 — Read the Plan: Key Node Types

| Node | Meaning | When it is a problem |
|------|---------|---------------------|
| `Seq Scan` | Full table scan | Large table (> 10MB) with high row selectivity |
| `Index Scan` | Random access via index | Usually good; high `rows removed by filter` is a warning |
| `Index Only Scan` | All data from index, no heap visit | Best case; requires vacuum to update visibility map |
| `Bitmap Index Scan` + `Bitmap Heap Scan` | Batched access, good for moderate selectivity | Check for `Lossy` in Bitmap Heap Scan |
| `Hash Join` | Build in-memory hash table for equi-join | If `Batches > 1`, it spilled to disk — increase `work_mem` |
| `Nested Loop` | For each outer row, scan inner side | Terrible on large outer sets without index on inner |
| `Merge Join` | Sort both sides then merge | Requires sorted input; can be slow if sort is needed |
| `Sort` | Explicit sort step | Check if an index on the ORDER BY column would eliminate it |
| `Hash Aggregate` | GROUP BY using hash table | Spill to disk if `Batches > 1` |
| `Gather` / `Gather Merge` | Parallel query node | Normal in parallel plans |

### Warning Signs to Look For

- **Rows estimated vs actual mismatch > 10x** — stale statistics; run `ANALYZE tablename`
- **Seq Scan on a large table** — missing index or `enable_seqscan = on` overriding
- **Sort Method: external merge Disk: N kB** — sort spilled to disk; increase `work_mem`
- **Hash Batches: N (original N)** where N > 1 — hash join spilled; increase `work_mem`
- **Rows Removed by Filter: N** (high) after an Index Scan — add a partial index or better column selection
- **Nested Loop** with large outer set — missing index on inner table join column

### Example Plan Analysis

```
Nested Loop  (cost=0.56..45231.82 rows=12 width=184)
             (actual time=0.095..8423.112 rows=12 loops=1)
  ->  Index Scan using orders_pkey on orders  (cost=0.43..8.45 rows=1 width=100)
                                              (actual time=0.042..0.044 rows=1 loops=12)
  ->  Seq Scan on order_items  (cost=0.00..4520.00 rows=8 width=84)
                               (actual time=701.900..701.908 rows=1 loops=12)
        Filter: (order_id = orders.id)
        Rows Removed by Filter: 90000
```

**Reading this:** The inner `Seq Scan on order_items` executes 12 times and each time reads 90,000 rows to find 1. Total: 1,080,000 rows scanned unnecessarily.

**Fix:** `CREATE INDEX ON order_items (order_id);`

---

## Step 4 — Index Type Selection

| Data type or query pattern | Index type | Syntax example |
|---------------------------|-----------|---------------|
| Equality, range, ORDER BY, LIKE 'prefix%' | B-tree (default) | `CREATE INDEX ON t (col)` |
| Equality only (UUID, integer primary lookups) | Hash | `CREATE INDEX ON t USING HASH (col)` |
| JSONB containment (`@>`), arrays (`&&`, `@>`) | GIN | `CREATE INDEX ON t USING GIN (payload)` |
| Full-text search | GIN | `CREATE INDEX ON t USING GIN (to_tsvector('english', body))` |
| Geometric, range types (tsrange, daterange), PostGIS | GiST | `CREATE INDEX ON t USING GIST (location)` |
| Very large append-only tables, sequential date columns | BRIN | `CREATE INDEX ON t USING BRIN (created_at)` |
| Filtered subset of rows | Partial B-tree | `CREATE INDEX ON t (col) WHERE status = 'active'` |
| Multiple columns in WHERE clause | Composite B-tree | `CREATE INDEX ON t (col1, col2)` |
| Cover all columns in SELECT (avoid heap access) | Covering with INCLUDE | `CREATE INDEX ON t (col1) INCLUDE (col2, col3)` |

### Creating Indexes Without Blocking Writes

```sql
-- Standard: locks out writers during build
CREATE INDEX idx_orders_customer ON orders (customer_id);

-- Non-blocking: takes longer but does not block writes
CREATE INDEX CONCURRENTLY idx_orders_customer ON orders (customer_id);

-- Rebuild a bloated index online
REINDEX INDEX CONCURRENTLY idx_orders_customer;
```

### Partial Index Examples

```sql
-- Only index active orders (smaller index, faster queries on active orders)
CREATE INDEX idx_active_orders ON orders (customer_id)
WHERE status = 'active';
-- Query MUST include: WHERE status = 'active' for the planner to use this index

-- Only index non-null emails
CREATE INDEX idx_users_email ON users (email)
WHERE email IS NOT NULL;

-- Partial index for a hot lookup pattern
CREATE INDEX idx_pending_jobs ON jobs (created_at)
WHERE processed = false;
```

### Covering Index (Enables Index Only Scan)

```sql
-- Query: SELECT status, total FROM orders WHERE customer_id = 42 ORDER BY order_date
-- Without INCLUDE: requires heap access for status and total
CREATE INDEX idx_orders_covering ON orders (customer_id, order_date)
INCLUDE (status, total);
-- Now the query can be served entirely from the index
```

---

## Step 5 — Statistics and the Planner

Incorrect row estimates are the most common cause of bad plans. Always check estimated vs actual rows in `EXPLAIN ANALYZE` output.

### Update Statistics

```sql
ANALYZE tablename;                -- Analyze one table
ANALYZE VERBOSE tablename;       -- With progress output
VACUUM ANALYZE tablename;        -- Vacuum + analyze together
```

### Increase Statistics Target for a Column

When a column has skewed or complex distribution, the default `statistics_target = 100` may produce poor estimates.

```sql
-- Check current statistics
SELECT attname, n_distinct, correlation, null_frac,
       most_common_vals, histogram_bounds
FROM pg_stats
WHERE tablename = 'orders' AND attname = 'status';

-- Increase statistics target for a column with complex distribution
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 500;
ANALYZE orders;
```

### Extended Statistics for Correlated Columns

```sql
-- When two columns are correlated (e.g., city and zip_code)
-- the planner underestimates combined selectivity
CREATE STATISTICS orders_stats ON customer_id, order_date FROM orders;
ANALYZE orders;

-- View statistics used by planner
SELECT * FROM pg_statistic_ext_data;
```

---

## Step 6 — Key Configuration Parameters

| Parameter | Default | Recommendation | Effect |
|-----------|---------|---------------|--------|
| `random_page_cost` | 4.0 | **1.1 for SSD**, 4.0 for HDD | Controls planner's preference for index vs seq scan. Lower = prefer indexes. |
| `effective_cache_size` | 4GB | 50–75% of RAM | Tells planner how much the OS caches. Does not allocate memory. |
| `work_mem` | 4MB | 16–256MB (OLTP), up to 1GB (analytics) | Per-sort-node, per-hash-node. Too low causes disk spill. |
| `effective_io_concurrency` | 1 | **100–200 for SSD** | Allows async prefetch in Bitmap Heap Scan. |
| `default_statistics_target` | 100 | 200–500 for complex queries | More samples = better estimates, slower ANALYZE. |
| `max_parallel_workers_per_gather` | 2 | 2–4 | Parallel query degree. 0 = disable parallel. |
| `jit` | on | **off for OLTP** | JIT compilation helps long analytics queries; adds overhead for short OLTP queries. |

### Changing a Parameter for Testing

```sql
-- Test in current session only (no restart needed)
SET work_mem = '256MB';
SET random_page_cost = 1.1;

-- Persist for a specific role
ALTER ROLE app_user SET work_mem = '64MB';
ALTER ROLE app_user SET random_page_cost = 1.1;

-- Persist globally (writes to postgresql.auto.conf)
ALTER SYSTEM SET random_page_cost = 1.1;
SELECT pg_reload_conf();
```

### Force/Disable Plan Choices for Debugging

```sql
-- Force the planner to avoid a seq scan (testing only)
SET enable_seqscan = off;
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;
RESET enable_seqscan;

-- Disable hash join to force merge or nested loop
SET enable_hashjoin = off;
```

---

## Common Query Rewrites

### Avoid Functions on Indexed Columns

```sql
-- Bad: function call on created_at prevents index use
SELECT * FROM orders WHERE date_trunc('day', created_at) = '2025-01-01';

-- Good: range query uses index
SELECT * FROM orders
WHERE created_at >= '2025-01-01' AND created_at < '2025-01-02';
```

### Keyset Pagination (Replace OFFSET)

```sql
-- Bad: OFFSET 1000000 scans and discards 1M rows
SELECT * FROM orders ORDER BY id OFFSET 1000000 LIMIT 10;

-- Good: cursor-based pagination using last seen ID
SELECT * FROM orders WHERE id > :last_seen_id ORDER BY id LIMIT 10;
```

### JSONB Query Optimization

```sql
-- Containment query: use GIN index
CREATE INDEX idx_events_gin ON events USING GIN (payload);
SELECT * FROM events WHERE payload @> '{"type": "login"}';

-- Specific key extraction: use expression index
CREATE INDEX idx_event_type ON events ((payload->>'type'));
SELECT * FROM events WHERE payload->>'type' = 'login';
```

---

## Reference Files

- [EXPLAIN ANALYZE guide](references/explain-guide.md) — all node types with good/bad indicators
- [Index types reference](references/index-types.md) — B-tree, Hash, GIN, GiST, BRIN with syntax and use cases
- [Planner configuration](references/planner-config.md) — all planner parameters, effects, and recommended values

## Examples

- [Optimization scenarios](examples/examples.md) — sequential scan fix, JSONB GIN index, partial index, statistics fix, pg_stat_statements analysis
