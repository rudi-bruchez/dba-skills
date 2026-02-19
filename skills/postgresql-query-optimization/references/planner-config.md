# PostgreSQL Planner Configuration Reference

## How the Planner Works

The PostgreSQL query planner is cost-based. It estimates the cost of every possible plan using:
1. **Table statistics** — row counts, column distributions, correlation
2. **Cost parameters** — configurable weights for I/O and CPU operations
3. **System parameters** — `effective_cache_size`, `work_mem`, etc.

The planner picks the lowest-cost plan. Wrong cost parameters or stale statistics cause bad plan choices.

---

## Cost Parameters

These parameters set the cost model weights. The unit is arbitrary; only ratios matter.

### random_page_cost and seq_page_cost

```ini
# Default (assumes spinning disk)
random_page_cost = 4.0
seq_page_cost = 1.0

# SSD storage (recommended)
random_page_cost = 1.1
seq_page_cost = 1.0
```

**Effect:** `random_page_cost` controls how expensive the planner thinks random I/O is relative to sequential I/O. At 4.0, the planner strongly prefers sequential scans over index scans for any query returning > ~2% of rows. At 1.1 (SSD), it much more readily uses indexes.

**This is the most impactful parameter to change on SSD storage.**

```sql
-- Set globally for SSD:
ALTER SYSTEM SET random_page_cost = 1.1;
SELECT pg_reload_conf();

-- Test effect on a specific query:
SET random_page_cost = 1.1;
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;
RESET random_page_cost;
```

### effective_io_concurrency

```ini
# Spinning disk: 2
# SSD: 100-200
# NVMe RAID: 300+
effective_io_concurrency = 200
```

**Effect:** Tells the planner how many concurrent disk reads are possible. Affects Bitmap Heap Scans (prefetching). Does not allocate any resources; it is purely a planning hint.

---

## Memory Parameters

### work_mem

```ini
work_mem = 64MB   # Per sort/hash operation (default: 4MB)
```

**Effect:** Each sort node and hash node can use up to `work_mem` before spilling to disk. A single query can use multiple sort/hash nodes simultaneously. A query with 3 hash joins can use up to `3 * work_mem`.

**Caution:** Total memory = `work_mem * active_queries * nodes_per_query`. With 100 connections and 4MB work_mem, that is already 400MB. Set conservatively for OLTP.

```sql
-- Check if queries are spilling (look for these in EXPLAIN output):
-- "Sort Method: external merge  Disk: 4096kB"
-- "Batches: 4  Memory Usage: 8192kB"  (Hash Join with 4 batches = spilled)

-- Set per-session for a specific complex query:
SET work_mem = '256MB';
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;
RESET work_mem;

-- Set per-role for reporting users:
ALTER ROLE reporter SET work_mem = '512MB';
```

**Typical values:**
- OLTP applications: 16–64MB
- Mixed workload: 64–256MB
- Reporting/analytics: 256MB–2GB

### effective_cache_size

```ini
effective_cache_size = 24GB   # 50-75% of total RAM (does NOT allocate memory)
```

**Effect:** Tells the planner how much total memory is available for caching (PostgreSQL buffer pool + OS page cache combined). Higher values make the planner more willing to use index scans. Does not allocate any memory; it is a planning hint only.

```sql
SHOW effective_cache_size;
-- Set to 50-75% of total system RAM:
ALTER SYSTEM SET effective_cache_size = '24GB';
SELECT pg_reload_conf();
```

### maintenance_work_mem

```ini
maintenance_work_mem = 1GB   # For VACUUM, CREATE INDEX, pg_dump
```

**Effect:** Memory for maintenance operations. Higher values allow `CREATE INDEX` to build indexes faster and `VACUUM` to collect more dead tuples per cycle.

---

## Statistics Parameters

### default_statistics_target

```ini
default_statistics_target = 100   # Default: samples 300 * 100 = 30,000 rows per column
```

**Effect:** Higher values produce more accurate estimates at the cost of longer ANALYZE time. A target of 200 gives more histogram buckets for columns with complex distributions.

```sql
-- Increase globally for complex workloads:
ALTER SYSTEM SET default_statistics_target = 200;
SELECT pg_reload_conf();
ANALYZE;  -- Re-analyze to use new target

-- Override for a specific column:
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 500;
ANALYZE orders (status);

-- Check current statistics quality:
SELECT attname, n_distinct, correlation, null_frac,
       array_length(most_common_vals::text::text[], 1) AS mcv_count,
       array_length(histogram_bounds::text::text[], 1) AS histogram_buckets
FROM pg_stats
WHERE tablename = 'orders';
```

### Extended Statistics

```sql
-- When two columns are correlated (planner underestimates combined selectivity)
-- Example: state and city are correlated (only certain combinations exist)
CREATE STATISTICS orders_location_stats ON state, city FROM customers;
ANALYZE customers;

-- Verify statistics were collected:
SELECT stxname, stxkind FROM pg_statistic_ext;

-- With explicit kinds:
CREATE STATISTICS orders_stats (ndistinct, dependencies, mcv)
    ON customer_id, order_date FROM orders;
```

---

## Parallel Query Configuration

```ini
max_worker_processes = 8           # Total background workers
max_parallel_workers = 8           # Max parallel query workers total
max_parallel_workers_per_gather = 4 # Max workers per query node
max_parallel_maintenance_workers = 4 # For CREATE INDEX, VACUUM
min_parallel_table_scan_size = 8MB  # Min table size to consider parallel scan
min_parallel_index_scan_size = 512kB # Min index size for parallel index scan
parallel_setup_cost = 1000         # Cost to launch a parallel worker
parallel_tuple_cost = 0.1          # Cost per tuple transferred between workers
```

**Effect:** The planner considers parallel plans when tables are large enough and parallelism cost is justified. Increase `max_parallel_workers_per_gather` to 4 for OLAP workloads; keep at 2 for OLTP.

```sql
-- Disable parallel for testing (session-level):
SET max_parallel_workers_per_gather = 0;

-- Force parallel for testing:
SET debug_parallel_query = on;   -- PG16+
-- SET force_parallel_mode = on;  -- Pre-PG16
```

---

## Enable/Disable Plan Types (Debugging Only)

These parameters force or prevent specific plan types. Use only for debugging — never leave them disabled in production.

```sql
-- Disable sequential scans (forces planner to find an index)
SET enable_seqscan = off;

-- Disable nested loop joins
SET enable_nestloop = off;

-- Disable hash joins
SET enable_hashjoin = off;

-- Disable merge joins
SET enable_mergejoin = off;

-- Disable index scans (force seq scan)
SET enable_indexscan = off;

-- Disable index-only scans
SET enable_indexonlyscan = off;

-- Disable bitmap index scans
SET enable_bitmapscan = off;

-- After testing, always reset:
RESET enable_seqscan;
RESET enable_nestloop;
-- or reset all at once:
RESET ALL;
```

**Workflow:** If `SET enable_seqscan = off` makes a query faster, it means there is an index the planner is not choosing. Investigate why: is `random_page_cost` too high? Are statistics stale? Is the index missing?

---

## JIT Compilation

```ini
jit = on                    # Default: on in PG15+
jit_above_cost = 100000     # Only JIT queries more expensive than this
jit_inline_above_cost = 500000
jit_optimize_above_cost = 500000
```

**Effect:** JIT compiles parts of the query into native code. Helps long-running analytical queries but adds overhead for short OLTP queries.

```sql
-- Disable JIT for an OLTP database:
ALTER SYSTEM SET jit = off;
SELECT pg_reload_conf();

-- Or disable for a specific role:
ALTER ROLE oltp_app SET jit = off;

-- Check if JIT was used in a plan:
EXPLAIN (ANALYZE, FORMAT TEXT) SELECT ...;
-- Look for: "JIT: Functions: N, Options: ..."
```

---

## Join Collapse and Order Preservation

```ini
join_collapse_limit = 8      # Default: reorder up to 8-table joins
from_collapse_limit = 8      # Default: flatten up to 8-item FROM clause
geqo_threshold = 12          # Use genetic algorithm for joins > 12 tables
```

```sql
-- Preserve explicit JOIN order (for performance testing):
SET join_collapse_limit = 1;
-- Now: SELECT * FROM a JOIN b ON ... JOIN c ON ...
-- Planner will use exactly this join order

-- Reset to default:
RESET join_collapse_limit;
```

---

## CTE Behavior (PostgreSQL 12+)

Pre-PG12: CTEs were always optimization fences (materialized before parent query sees them).
PG12+: CTEs are inlined by default unless they are recursive or have side effects.

```sql
-- Force materialization (old behavior, isolates subquery):
WITH expensive_cte AS MATERIALIZED (
    SELECT * FROM large_table WHERE ...
)
SELECT * FROM expensive_cte WHERE additional_filter;

-- Force inlining (allows predicate pushdown):
WITH cte AS NOT MATERIALIZED (
    SELECT * FROM large_table
)
SELECT * FROM cte WHERE ...;
```

---

## Configuration Templates

### OLTP Server (SSD storage)
```ini
random_page_cost = 1.1
effective_io_concurrency = 200
work_mem = 32MB
effective_cache_size = 24GB    # 75% of 32GB RAM
default_statistics_target = 100
max_parallel_workers_per_gather = 2
jit = off
```

### Analytics / Reporting Server (SSD)
```ini
random_page_cost = 1.1
effective_io_concurrency = 200
work_mem = 1GB
maintenance_work_mem = 4GB
effective_cache_size = 96GB    # 75% of 128GB RAM
default_statistics_target = 500
max_parallel_workers_per_gather = 16
max_parallel_workers = 32
jit = on
jit_above_cost = 50000
enable_partitionwise_join = on
enable_partitionwise_aggregate = on
```

### Spinning Disk Server
```ini
random_page_cost = 4.0
effective_io_concurrency = 2
work_mem = 16MB
effective_cache_size = 12GB
default_statistics_target = 100
max_parallel_workers_per_gather = 2
jit = off
```
