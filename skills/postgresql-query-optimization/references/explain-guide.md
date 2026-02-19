# EXPLAIN ANALYZE Guide

## Running EXPLAIN ANALYZE

```sql
-- Basic: execute query, show actual vs estimated
EXPLAIN (ANALYZE) SELECT ...;

-- Recommended: include buffer statistics
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;

-- For write statements: wrap in transaction
BEGIN;
EXPLAIN (ANALYZE, BUFFERS) UPDATE orders SET status = 'done' WHERE id = 42;
ROLLBACK;

-- High-traffic queries: disable per-node timing to reduce overhead
EXPLAIN (ANALYZE, BUFFERS, TIMING OFF) SELECT ...;

-- Machine-readable JSON for tools
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) SELECT ...;

-- Show planner settings used (PG12+)
EXPLAIN (ANALYZE, VERBOSE, SETTINGS) SELECT ...;

-- Include WAL usage (PG13+)
EXPLAIN (ANALYZE, WAL) UPDATE ...;
```

---

## Plan Structure

Plans are trees. The innermost (most-indented) nodes execute first. Each node feeds rows up to its parent.

```
Hash Join  (cost=100.0..500.0 rows=1000 width=50)
           (actual time=5.2..12.1 rows=985 loops=1)
  Buffers: shared hit=234 read=45
  ->  Seq Scan on customers  (cost=0.0..200.0 rows=10000 width=30)
                             (actual time=0.01..2.5 rows=10000 loops=1)
        Buffers: shared hit=180
  ->  Hash  (cost=50.0..50.0 rows=4000 width=20)
            (actual time=2.1..2.1 rows=4000 loops=1)
        Buckets: 4096  Batches: 1  Memory Usage: 256kB
        ->  Seq Scan on orders  (cost=0.0..50.0 rows=4000 width=20)
                               (actual time=0.01..1.1 rows=4000 loops=1)

Planning Time: 0.5 ms
Execution Time: 12.8 ms
```

---

## Reading Cost and Row Estimates

```
(cost=start..total rows=N width=W)
(actual time=start..total rows=N loops=L)
```

- **cost=start..total** — Planner's cost estimate in arbitrary units (not milliseconds)
- **rows** — Estimated row count from planner
- **width** — Estimated average row width in bytes
- **actual time** — Real wall-clock time in milliseconds
- **actual rows** — Real rows returned **per loop**
- **loops** — How many times this node executed
- **Total actual rows** = `actual rows * loops`

**Critical check:** Compare estimated rows to actual rows. A mismatch > 10x means stale or inadequate statistics.

---

## All Node Types

### Scan Nodes

#### Seq Scan
```
Seq Scan on orders  (cost=0.0..4532.0 rows=180000 width=48)
                   (actual time=0.01..23.4 rows=180000 loops=1)
  Filter: (status = 'active')
  Rows Removed by Filter: 120000
```
- **What it does:** Reads every row in the table sequentially.
- **When it is good:** Small tables (< 1MB), or query returns > 10–20% of rows.
- **When it is bad:** Large tables with a selective filter. A missing index is the usual cause.
- **Fix:** Create an index on the filter column. If `Rows Removed by Filter` is large and the condition is selective, create a partial index.

#### Index Scan
```
Index Scan using orders_customer_id_idx on orders
    (cost=0.43..8.61 rows=1 width=48)
    (actual time=0.04..0.05 rows=1 loops=1)
  Index Cond: (customer_id = 42)
```
- **What it does:** Uses an index to find row locations, then fetches rows from the heap (table).
- **When it is good:** Selective queries (< 5–10% of rows). Each row fetch is a random I/O.
- **When it is bad:** On very large result sets, the random I/O overhead exceeds a seq scan. PostgreSQL usually switches to Bitmap Scan or Seq Scan automatically.
- **Warning sign:** `Filter: (condition)` after `Index Cond` means the index is not selective enough — consider a composite or partial index.

#### Index Only Scan
```
Index Only Scan using orders_covering_idx on orders
    (cost=0.43..4.45 rows=1 width=20)
    (actual time=0.02..0.03 rows=1 loops=1)
  Index Cond: (customer_id = 42)
  Heap Fetches: 0
```
- **What it does:** All required columns are in the index; no heap visit needed.
- **When it is good:** `Heap Fetches: 0` — ideal, all data from index.
- **When it is bad:** `Heap Fetches: N` (high) means the visibility map is not up to date. Run `VACUUM` to update it.
- **How to enable:** Use `INCLUDE` columns in index: `CREATE INDEX ON orders (customer_id) INCLUDE (status, total);`

#### Bitmap Index Scan + Bitmap Heap Scan
```
Bitmap Heap Scan on orders  (cost=95.5..423.8 rows=500 width=48)
                           (actual time=1.2..4.5 rows=487 loops=1)
  Recheck Cond: (customer_id = 42)
  Heap Blocks: exact=320
  ->  Bitmap Index Scan on orders_customer_idx
          (cost=0.0..95.4 rows=500 width=0)
          (actual time=0.8..0.8 rows=487 loops=1)
        Index Cond: (customer_id = 42)
```
- **What it does:** First pass builds a bitmap of matching pages; second pass fetches all rows from those pages in page order (fewer random I/Os than Index Scan for large result sets).
- **When it is good:** Moderate selectivity (hundreds to thousands of rows).
- **Warning sign:** `Heap Blocks: lossy=N` means the bitmap exceeded `work_mem` and became page-level (less precise, requiring recheck). Increase `work_mem`.
- **Multiple conditions:** PostgreSQL can combine multiple Bitmap Index Scans with BitmapAnd/BitmapOr.

### Join Nodes

#### Hash Join
```
Hash Join  (cost=1000.0..5000.0 rows=10000 width=60)
          (actual time=12.3..45.2 rows=9800 loops=1)
  Hash Cond: (orders.customer_id = customers.id)
  Buffers: shared hit=1200 read=300
  ->  Seq Scan on orders  ...
  ->  Hash  (cost=500.0..500.0 rows=50000 width=30)
            (actual time=8.1..8.1 rows=50000 loops=1)
        Buckets: 65536  Batches: 1  Memory Usage: 3200kB
        ->  Seq Scan on customers  ...
```
- **What it does:** Builds an in-memory hash table from the smaller relation, then probes it for each row from the larger relation.
- **When it is good:** Large equi-joins where neither side is indexed.
- **Warning sign:** `Batches: N > 1` means the hash table spilled to disk. Increase `work_mem`.
- **Memory used:** `Memory Usage: NkB` — if this approaches `work_mem`, add more.

#### Nested Loop
```
Nested Loop  (cost=0.43..245.0 rows=10 width=60)
            (actual time=0.08..0.95 rows=10 loops=1)
  ->  Index Scan using customers_pkey on customers  (loops=1, rows=10)
  ->  Index Scan using orders_customer_id on orders  (loops=10, rows=1)
```
- **What it does:** For each row from the outer side, scans the inner side.
- **When it is good:** Outer set is small and inner has an index.
- **When it is bad:** Large outer set. `loops=10000` with a slow inner scan = disaster.
- **Fix:** Ensure the inner join column is indexed. If outer set is large, switching to Hash Join may help (`SET enable_nestloop = off` to test).

#### Merge Join
```
Merge Join  (cost=1234.5..2000.0 rows=5000 width=60)
           (actual time=5.2..18.4 rows=5000 loops=1)
  Merge Cond: (orders.customer_id = customers.id)
  ->  Sort  (cost=500.0..512.5 rows=5000 width=30)
            (actual time=3.1..3.4 rows=5000 loops=1)
        Sort Key: orders.customer_id
  ->  Index Scan using customers_pkey on customers  ...
```
- **What it does:** Both sides must be sorted on join key; then merges them in one pass.
- **When it is good:** Both sides already sorted (e.g., via index); large joins.
- **Warning sign:** Sort step before Merge Join is expensive. An index that provides sorted output eliminates the Sort node.

### Aggregate and Sort Nodes

#### Sort
```
Sort  (cost=1234.0..1259.0 rows=10000 width=48)
     (actual time=15.3..16.8 rows=10000 loops=1)
  Sort Key: created_at DESC
  Sort Method: external merge  Disk: 4096kB
```
- **When good:** Small sorts in memory (`Sort Method: quicksort` or `top-N heapsort`).
- **Warning sign:** `Sort Method: external merge  Disk: NkB` — sort spilled to disk. Increase `work_mem` or add an index on the ORDER BY column.

#### Hash Aggregate
```
HashAggregate  (cost=500.0..510.0 rows=100 width=16)
              (actual time=5.2..5.4 rows=100 loops=1)
  Group Key: status
  Batches: 1  Memory Usage: 24kB
```
- **Warning sign:** `Batches: N > 1` — aggregate spilled to disk. Increase `work_mem`.

---

## Buffer Statistics

With `BUFFERS` option, each node shows:
```
Buffers: shared hit=234 read=45 dirtied=0 written=0
```

| Field | Meaning |
|-------|---------|
| `shared hit` | Pages found in PostgreSQL's shared buffer cache |
| `shared read` | Pages read from OS/disk |
| `shared dirtied` | Pages modified in cache |
| `shared written` | Pages written to disk from cache |
| `local hit/read` | Temporary table buffers |
| `temp read/written` | Temp files (sort/hash spills) |

**Cache hit ratio per node:** `hit / (hit + read)` should be > 95% for hot data.

**High `read` count** on a specific node indicates that table or index is not in cache — either `shared_buffers` is too small or the data is too large to cache.

---

## Planning Time vs Execution Time

```
Planning Time: 0.8 ms
Execution Time: 125.4 ms
```

- **High planning time** (> 5ms for simple queries): Too many tables in join, excessive OR conditions, or planner genetic algorithm kicking in (`geqo_threshold`).
- **Low execution time despite slow response**: Network overhead or result serialization is the bottleneck.
