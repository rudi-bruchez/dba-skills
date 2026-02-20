# PostgreSQL Query Optimization Examples

Practical scenarios and how to optimize slow PostgreSQL queries.

## Scenario 1: Identifying Top CPU Queries

### Diagnosis
1.  **Top Queries:** Use `pg_stat_statements` to find the most resource-intensive queries.
```sql
SELECT query, calls, total_exec_time, mean_exec_time, rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 5;
```
2.  **Observation:** A query on the `orders` table is taking several seconds and has a high `mean_exec_time`.
3.  **Action:** Run `EXPLAIN (ANALYZE, BUFFERS)` on the query.
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE customer_id = 12345;
```
4.  **Observation:** High-cost "Seq Scan" (Full Table Scan) on `orders`.
5.  **Root Cause:** No index on the `customer_id` column.

### Optimization
Create a B-tree index on `customer_id`.
```sql
CREATE INDEX idx_orders_customer_id ON orders (customer_id);
```
**Result:** The execution plan now uses an "Index Scan" or "Index Only Scan," significantly reducing the execution time.

## Scenario 2: Indexing for JSONB Data

### Diagnosis
1.  **Query:** Searching for a specific value in a `JSONB` column.
```sql
SELECT * FROM users WHERE data->>'role' = 'admin';
```
2.  **Execution Plan:** High-cost "Seq Scan" because B-tree indexes don't support `JSONB` path searches directly.

### Optimization
Create a GIN index on the `JSONB` column.
```sql
CREATE INDEX idx_users_data_gin ON users USING GIN (data jsonb_path_ops);
```
Rewrite the query to use the `contains (@>)` operator.
```sql
SELECT * FROM users WHERE data @> '{"role": "admin"}';
```
**Result:** The execution plan now uses a "Bitmap Index Scan" on `idx_users_data_gin`, which is much faster than a sequential scan.

## Scenario 3: Non-SARGable Queries

### Diagnosis
1.  **Query:** Filtering by a transformed column value.
```sql
SELECT * FROM users WHERE LOWER(email) = 'rudi@example.com';
```
2.  **Execution Plan:** High-cost "Seq Scan" even if an index exists on `email`.

### Optimization
Create an expression index on `LOWER(email)`.
```sql
CREATE INDEX idx_users_email_lower ON users (LOWER(email));
```
**Result:** The execution plan now uses an "Index Scan" on `idx_users_email_lower`.

## Scenario 4: Optimizing Joins with Work Mem

### Diagnosis
1.  **Query:** Joining two large tables.
```sql
SELECT o.order_id, c.customer_name
FROM orders AS o
JOIN customers AS c ON o.customer_id = c.customer_id;
```
2.  **Execution Plan:** A "Hash Join" is used, but it's slow.
3.  **Observation:** `EXPLAIN (ANALYZE, BUFFERS)` shows `Hash Join` with `Buckets: 1024 Batches: 16 Memory Usage: 1024kB`. The "Batches > 1" indicates that the hash table spilled to disk.

### Optimization
Increase `work_mem` for the current session.
```sql
SET work_mem = '64MB';
```
Run the query again.
```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;
```
**Observation:** `Hash Join` now has `Batches: 1`, indicating that the hash table fit in memory.
**Result:** Significant performance improvement by avoiding slow disk writes during the join.
