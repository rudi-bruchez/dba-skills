# PostgreSQL Query Optimization & Performance Tuning

## Overview
Query optimization in PostgreSQL involves understanding the query planner, using EXPLAIN/EXPLAIN ANALYZE, choosing appropriate indexes, managing statistics, and tuning configuration parameters that affect plan choices. The planner is cost-based; understanding its cost model is essential for effective tuning.

---

## EXPLAIN & Query Plan Analysis

### EXPLAIN Options
```sql
EXPLAIN SELECT ...;                          -- Show plan with estimated costs
EXPLAIN (ANALYZE) SELECT ...;               -- Execute and show actual vs estimated
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;      -- Add buffer hit/miss stats
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) SELECT ...; -- Machine-readable output
EXPLAIN (ANALYZE, BUFFERS, TIMING OFF) SELECT ...; -- Reduce timing overhead
EXPLAIN (ANALYZE, VERBOSE, SETTINGS) SELECT ...; -- Include GUC settings used (PG12+)
EXPLAIN (ANALYZE, WAL) SELECT ...;          -- Include WAL usage (PG13+)
EXPLAIN (GENERIC_PLAN) SELECT $1 ...;       -- Show generic plan (PG16+)
```

### Reading EXPLAIN Output — Key Fields
- **cost=start..total** — Planner's estimate in arbitrary cost units (not ms)
- **rows** — Estimated row count (compare to actual)
- **width** — Estimated average row width in bytes
- **actual time=start..total** — Real execution time in ms (with ANALYZE)
- **actual rows** — Real rows returned
- **loops** — Number of times this node executed; actual rows is per loop
- **Buffers: shared hit=N read=N** — Cache hits vs disk reads
- **Planning Time** — Time to generate the plan
- **Execution Time** — Total query execution time

### Plan Node Types to Know
| Node | Meaning |
|------|---------|
| `Seq Scan` | Full table scan; normal for small tables or large % of rows |
| `Index Scan` | Random access via index; good for selective queries |
| `Index Only Scan` | Data served entirely from index; check visibility map |
| `Bitmap Index Scan` + `Bitmap Heap Scan` | Batched random I/O; good for moderate selectivity |
| `Hash Join` | Build hash table from smaller relation; good for large equi-joins |
| `Nested Loop` | For each row in outer, scan inner; good when inner is indexed |
| `Merge Join` | Sort both sides then merge; good for sorted or indexed data |
| `Sort` | Explicit sort — check if index can eliminate this |
| `Hash Aggregate` | Hash-based GROUP BY |
| `Gather` / `Gather Merge` | Parallel query coordination |

### Warning Signs in Plans
- **Rows estimated vs actual mismatch > 10x** — stale or missing statistics
- **Seq Scan on large table** — missing index or planner choosing not to use it
- **Hash Join spilling to disk** — increase `work_mem`
- **Sort spilling to disk** — increase `work_mem`
- **Nested Loop with large outer set** — consider join order hints or indexes
- **Filter: (condition)** after an Index Scan — partial index or better index needed
- **Rows Removed by Filter: N** — high numbers indicate poor index selectivity

---

## Statistics & the Query Planner

### Key Statistics Tables
```sql
-- Column-level statistics
SELECT attname, n_distinct, correlation, null_frac, avg_width,
       most_common_vals, most_common_freqs, histogram_bounds
FROM pg_stats
WHERE tablename = 'mytable';

-- Table-level statistics
SELECT relname, reltuples, relpages, relallvisible
FROM pg_class
WHERE relname = 'mytable';
```

### Updating Statistics
```sql
ANALYZE mytable;                        -- Analyze one table
ANALYZE VERBOSE mytable;                -- With progress output
ANALYZE mytable (col1, col2);           -- Specific columns (PG14+)
VACUUM ANALYZE mytable;                 -- Combined vacuum + analyze
```

### Increasing Statistics Target
```sql
-- Default statistics_target = 100 (samples up to 300 * target rows)
ALTER TABLE mytable ALTER COLUMN status SET STATISTICS 500;
-- Or globally:
SET default_statistics_target = 200;    -- Session-level
-- In postgresql.conf: default_statistics_target = 200
```

### Extended Statistics (Multi-Column)
```sql
-- Capture correlation between columns (PG10+)
CREATE STATISTICS mystat ON col1, col2 FROM mytable;
CREATE STATISTICS mystat (ndistinct, dependencies) ON col1, col2 FROM mytable;
ANALYZE mytable;

-- View extended stats
SELECT * FROM pg_statistic_ext_data;
```

---

## Index Types & Usage

### B-Tree (Default)
```sql
CREATE INDEX idx_name ON table (col);
CREATE INDEX idx_name ON table (col1, col2);          -- Composite
CREATE INDEX idx_name ON table (col DESC NULLS LAST); -- Sort order matters
CREATE UNIQUE INDEX idx_name ON table (col);
```
- Best for: equality (`=`), range (`<`, `>`, `BETWEEN`), `ORDER BY`, `GROUP BY`
- Supports: IS NULL, IS NOT NULL
- Composite: leftmost prefix rule applies

### Partial Indexes
```sql
-- Only index active records
CREATE INDEX idx_active_orders ON orders (customer_id)
WHERE status = 'active';

-- Only index non-null values
CREATE INDEX idx_email ON users (email)
WHERE email IS NOT NULL;
```
- Smaller, faster to maintain, planner must recognize the condition matches

### Expression (Functional) Indexes
```sql
CREATE INDEX idx_lower_email ON users (lower(email));
-- Query must use the same expression:
SELECT * FROM users WHERE lower(email) = 'foo@example.com';

CREATE INDEX idx_year ON orders (extract(year FROM created_at));
```

### Hash Indexes (PG10+ WAL-logged)
```sql
CREATE INDEX idx_hash ON table USING HASH (col);
```
- Only for equality; smaller than B-tree for equality-only workloads

### GIN Indexes
```sql
-- Full-text search
CREATE INDEX idx_fts ON docs USING GIN (to_tsvector('english', content));
-- JSONB containment
CREATE INDEX idx_json ON events USING GIN (payload);
-- Array containment
CREATE INDEX idx_tags ON articles USING GIN (tags);
```

### GiST Indexes
```sql
-- Geometric types, range types, full-text
CREATE INDEX idx_geo ON locations USING GIST (coordinates);
CREATE INDEX idx_range ON bookings USING GIST (during);  -- tsrange/daterange
```

### BRIN Indexes
```sql
-- Very large, naturally ordered tables (e.g., time-series, append-only)
CREATE INDEX idx_brin ON events USING BRIN (created_at);
-- Much smaller than B-tree; less accurate, faster to build
```

### Index Maintenance Queries
```sql
-- Index bloat estimate
SELECT indexname,
       pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
       idx_scan,
       idx_tup_read,
       idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC;

-- Duplicate/redundant indexes
SELECT indrelid::regclass AS table,
       array_agg(indexrelid::regclass) AS indexes,
       array_agg(pg_get_indexdef(indexrelid)) AS defs
FROM pg_index
GROUP BY indrelid, indkey
HAVING count(*) > 1;

-- Rebuild an index online
REINDEX INDEX CONCURRENTLY idx_name;
REINDEX TABLE CONCURRENTLY mytable;
```

---

## Vacuuming & Table Maintenance

### VACUUM Variants
```sql
VACUUM mytable;                    -- Remove dead tuples, update visibility map
VACUUM VERBOSE mytable;            -- With progress detail
VACUUM ANALYZE mytable;            -- Vacuum + statistics update
VACUUM FULL mytable;               -- Full rewrite, reclaims space, locks table
VACUUM FREEZE mytable;             -- Force freeze all tuples (prevents wraparound)
```

### Autovacuum Tuning per Table
```sql
-- Override autovacuum thresholds for a specific table
ALTER TABLE mytable SET (
    autovacuum_vacuum_scale_factor = 0.01,    -- 1% dead tuples triggers vacuum (default 20%)
    autovacuum_analyze_scale_factor = 0.005,  -- 0.5% changes triggers analyze
    autovacuum_vacuum_cost_delay = 2,          -- Reduce I/O throttling (ms)
    autovacuum_vacuum_cost_limit = 400         -- More work per cycle
);

-- Reset to defaults
ALTER TABLE mytable RESET (autovacuum_vacuum_scale_factor);
```

### Checking Autovacuum Status
```sql
SELECT schemaname,
       relname,
       last_vacuum,
       last_autovacuum,
       last_analyze,
       last_autoanalyze,
       vacuum_count,
       autovacuum_count,
       n_dead_tup,
       n_mod_since_analyze,
       n_ins_since_vacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;

-- Tables that need manual vacuum (dead tuple ratio)
SELECT relname,
       n_dead_tup,
       n_live_tup,
       round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct,
       last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
  AND round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) > 5
ORDER BY dead_pct DESC;
```

---

## Query Tuning Techniques

### work_mem (Sort & Hash Join Memory)
```sql
-- Set for current session (affects each sort/hash node independently)
SET work_mem = '256MB';

-- For a specific query
SET LOCAL work_mem = '512MB';

-- Check how much was spilled to disk (look in EXPLAIN ANALYZE output)
-- "Sort Method: external merge  Disk: 12345kB" means spill occurred
```

### Join Order & Planning Hints
```sql
-- Disable specific join methods for testing
SET enable_hashjoin = off;
SET enable_nestloop = off;
SET enable_mergejoin = off;
SET enable_seqscan = off;     -- Force index scan for testing

-- Always re-enable after testing!
RESET enable_hashjoin;

-- Use join_collapse_limit and from_collapse_limit
SET join_collapse_limit = 1;   -- Preserve explicit JOIN order
```

### Parallel Query Configuration
```sql
-- Enable parallel query
SET max_parallel_workers_per_gather = 4;
SET parallel_tuple_cost = 0.1;
SET parallel_setup_cost = 1000;
SET min_parallel_table_scan_size = '8MB';
SET min_parallel_index_scan_size = '512kB';

-- Force parallel for testing
SET force_parallel_mode = on;   -- Deprecated in PG16, use debug_parallel_query

-- Check if query used parallel execution
-- Look for "Gather" or "Gather Merge" nodes in EXPLAIN output
```

### CTEs vs Subqueries
```sql
-- PG12+: CTEs are NOT optimization fences by default
-- Pre-PG12: CTEs were always materialized (optimization fences)

-- Force materialization (optimization fence behavior):
WITH cte AS MATERIALIZED (SELECT ...)
SELECT * FROM cte WHERE ...;

-- Force no materialization:
WITH cte AS NOT MATERIALIZED (SELECT ...)
SELECT * FROM cte WHERE ...;
```

### Covering Indexes (Index Only Scans)
```sql
-- Include non-key columns to enable Index Only Scans
CREATE INDEX idx_covering ON orders (customer_id, order_date)
INCLUDE (status, total_amount);
-- Query: SELECT status, total_amount FROM orders WHERE customer_id = 1
-- Can be served entirely from the index

-- Check visibility map for Index Only Scan effectiveness
SELECT relname,
       relallvisible,
       relpages,
       round(100.0 * relallvisible / NULLIF(relpages, 0), 1) AS pct_visible
FROM pg_class
WHERE relname = 'orders';
```

---

## Common Performance Patterns

### Pagination
```sql
-- Bad: OFFSET is slow for large pages
SELECT * FROM orders ORDER BY id OFFSET 1000000 LIMIT 10;

-- Good: Keyset pagination (cursor-based)
SELECT * FROM orders
WHERE id > :last_seen_id
ORDER BY id
LIMIT 10;
```

### EXISTS vs IN vs JOIN
```sql
-- EXISTS is often fastest for existence checks
SELECT * FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);

-- IN with subquery can be slow if subquery returns many rows
-- JOIN is equivalent but optimizer may handle differently
```

### Avoiding Function Calls on Indexed Columns
```sql
-- Bad: function call prevents index use
SELECT * FROM users WHERE date_trunc('day', created_at) = '2025-01-01';

-- Good: range condition uses index
SELECT * FROM users
WHERE created_at >= '2025-01-01' AND created_at < '2025-01-02';
```

### JSONB Query Optimization
```sql
-- Use GIN index for containment
CREATE INDEX idx_gin ON events USING GIN (payload);
SELECT * FROM events WHERE payload @> '{"type": "login"}';

-- Use expression index for specific key access
CREATE INDEX idx_event_type ON events ((payload->>'type'));
SELECT * FROM events WHERE payload->>'type' = 'login';
```

---

## Key Configuration Parameters for Query Performance

| Parameter | Default | Recommendation |
|-----------|---------|----------------|
| `shared_buffers` | 128MB | 25% of RAM |
| `work_mem` | 4MB | 4-64MB (multiply by max_connections * sorts) |
| `effective_cache_size` | 4GB | 50-75% of RAM (planner hint only) |
| `random_page_cost` | 4.0 | 1.1-2.0 for SSDs |
| `effective_io_concurrency` | 1 | 100-200 for SSDs |
| `default_statistics_target` | 100 | 200-500 for complex queries |
| `max_parallel_workers_per_gather` | 2 | 2-4 |
| `enable_partitionwise_join` | off | on for partitioned tables |
| `enable_partitionwise_aggregate` | off | on for partitioned tables |
| `jit` | on | off for OLTP, on for analytics |

---

## pg_stat_statements Analysis Queries

```sql
-- Most time-consuming queries
SELECT queryid,
       left(query, 120) AS query,
       calls,
       round(total_exec_time::numeric / 1000, 2) AS total_sec,
       round(mean_exec_time::numeric, 2) AS avg_ms,
       round(stddev_exec_time::numeric, 2) AS stddev_ms,
       rows / NULLIF(calls, 0) AS rows_per_call
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Highest I/O queries
SELECT queryid,
       left(query, 120) AS query,
       calls,
       shared_blks_read + local_blks_read AS total_reads,
       shared_blks_hit,
       round(100.0 * shared_blks_hit /
             NULLIF(shared_blks_hit + shared_blks_read, 0), 2) AS cache_hit_pct
FROM pg_stat_statements
ORDER BY (shared_blks_read + local_blks_read) DESC
LIMIT 10;

-- Most variable (inconsistent) queries
SELECT queryid,
       left(query, 120) AS query,
       calls,
       round(mean_exec_time::numeric, 2) AS avg_ms,
       round(stddev_exec_time::numeric, 2) AS stddev_ms,
       round(stddev_exec_time / NULLIF(mean_exec_time, 0), 2) AS coeff_variation
FROM pg_stat_statements
WHERE calls > 100
ORDER BY coeff_variation DESC
LIMIT 10;

-- Reset statistics
SELECT pg_stat_statements_reset();
```

---

## Partitioning for Performance

```sql
-- Range partitioning (common for time-series)
CREATE TABLE orders (
    id bigserial,
    created_at timestamptz NOT NULL,
    customer_id int,
    total numeric
) PARTITION BY RANGE (created_at);

CREATE TABLE orders_2025_01 PARTITION OF orders
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

-- List partitioning
CREATE TABLE orders_by_region (
    id bigserial,
    region text NOT NULL
) PARTITION BY LIST (region);

-- Hash partitioning (even distribution)
CREATE TABLE orders_sharded (
    id bigserial
) PARTITION BY HASH (id);

-- Enable partition pruning (default on)
SET enable_partition_pruning = on;
```

---

## Best Practices Summary

1. Always use `EXPLAIN (ANALYZE, BUFFERS)` — never just `EXPLAIN` for real workloads.
2. Compare **estimated vs actual rows** — mismatch > 10x means stale stats or missing extended stats.
3. Set `random_page_cost = 1.1` for SSD storage — critical for index usage decisions.
4. Use **partial indexes** aggressively for filtered queries (e.g., WHERE status = 'pending').
5. **`INCLUDE` columns** in indexes to enable Index Only Scans for high-frequency queries.
6. Monitor `pg_stat_statements` weekly and tune the top 10 queries by total time.
7. Avoid `SELECT *` — it prevents Index Only Scans and transfers unnecessary data.
8. Use **connection pooling** (PgBouncer) to keep `work_mem * max_connections` manageable.
9. Test plan changes with `SET enable_* = off` before adding hints or changing config.
10. **Partition large tables** (> 100M rows or > 100GB) for maintenance and query efficiency.
