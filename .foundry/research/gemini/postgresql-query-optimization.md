# PostgreSQL Query Optimization Research

## Overview
Query optimization in PostgreSQL involves analyzing and improving SQL query execution plans. The database uses a cost-based optimizer to select the most efficient plan.

## Query Analysis (Explain)
- **`EXPLAIN`:** Shows the execution plan.
- **`EXPLAIN (ANALYZE, BUFFERS)`:** Runs the query, providing actual execution times and data block (buffer) usage. This is the most important diagnostic tool.
- **Node Types:**
    - **Seq Scan:** Full table scan.
    - **Index Scan:** Scan of the index only.
    - **Index Only Scan:** Retrieves data directly from the index (no visit to the heap). Requires a clean "visibility map."
    - **Bitmap Scan:** A hybrid of sequential and index scans for large result sets.
    - **Nested Loop / Hash Join / Merge Join:** Different algorithms for joining tables.

## Indexing Strategies
- **B-tree Index:** The default and most common index type.
- **GIN (Generalized Inverted Index):** Optimized for multi-value columns (JSONB, arrays, full-text search).
- **GiST (Generalized Search Tree):** Used for geometric data, range types, and complex types.
- **BRIN (Block Range Index):** Extremely compact index for very large, naturally-ordered tables (e.g., log data).
- **Partial Indexes:** Indexing only a subset of rows (e.g., `WHERE status = 'pending'`).
- **Expression Indexes:** Indexing the result of a function (e.g., `LOWER(email)`).

## Server Configuration (Performance Parameters)
- **`shared_buffers`:** Memory for caching data. Recommended 25% of RAM.
- **`work_mem`:** Memory for sorts and hash joins. Allocated per operation. Set according to the query workload.
- **`maintenance_work_mem`:** Memory for maintenance tasks like `VACUUM` and `CREATE INDEX`.
- **`effective_cache_size`:** Informs the optimizer about total available cache (including OS cache). Recommended 50-75% of RAM.
- **`random_page_cost`:** Cost factor for random access vs. sequential access. For SSDs, set this to 1.1 or 1.0.

## Query Rewriting & Common Anti-Patterns
- **Avoid `SELECT *`:** Only fetch required columns.
- **Correlated Subqueries:** Replace with joins or CTEs when possible.
- **Common Table Expressions (CTEs):** In older versions (pre-PG 12), CTEs were an optimization fence. In PG 12+, they are inlined by default.
- **SARGability:** Ensure your `WHERE` clauses can use indexes (e.g., avoid `WHERE LOWER(name) = 'rudi'`; use an expression index or a case-insensitive collation).

## Best Practices (2024-2025)
- **`pg_stat_statements`:** Use this extension to identify the most resource-intensive queries across the entire instance.
- **Visibility Map:** Ensure autovacuum is keeping the visibility map clean to allow more Index Only Scans.
- **Partitioning:** Use declarative partitioning for large tables to allow partition pruning.
- **Parallel Query:** Tune `max_parallel_workers_per_gather` and `max_worker_processes` to leverage multiple CPU cores.

## Key SQL Queries
- `EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM table WHERE col = 'val';`
- `SELECT * FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 5;`

## References
- [PostgreSQL Documentation: Query Planning](https://www.postgresql.org/docs/current/planner-optimizer.html)
- [PostgresQLExplain.com: Visualizer for JSON plans](https://explain.depesz.com/)
- [pganalyze: Index Advisor](https://pganalyze.com/index-advisor)
