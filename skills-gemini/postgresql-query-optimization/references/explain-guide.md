# Explain Guide

The `EXPLAIN` command shows the execution plan for a PostgreSQL query. Adding `ANALYZE` and `BUFFERS` provides the actual execution times and data block usage.

## Key Node Types

| Node Type | Description | Warning Threshold |
|---|---|---|
| **Seq Scan** | Sequential (full) table scan. Reads all pages in the table. | Inefficient for large tables. Create an index! |
| **Index Scan** | Navigates the index to find matching rows, then fetches them from the table. | Most efficient access method for a small range of rows. |
| **Index Only Scan**| Retrieves all data directly from the index (no visit to the table heap). | Requires a clean "visibility map" (maintained by autovacuum). |
| **Bitmap Index Scan**| Hybrid of Seq Scan and Index Scan for large result sets. | Good for many matching rows; avoids random I/O. |
| **Nested Loop Join** | For each row in the outer table, search for a match in the inner table. | Inefficient for large inner tables without a good index. |
| **Hash Join** | Builds a hash table in memory to perform the join. | Efficient for large result sets. May "spill" to disk if `work_mem` is insufficient. |
| **Merge Join** | Sorts both tables (if needed) and merges them. | Efficient for large, already-sorted datasets. |
| **Sort** | Sorts the input data. | Sorting should be avoided if possible by using indexes that provide the requested order. |

## Explain (ANALYZE, BUFFERS) Example

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM users WHERE email = 'rudi@example.com';
```

Output highlights:
- **Actual time:** The real-time taken by each node.
- **Buffers: shared hit=X read=Y:** How many data blocks were found in `shared_buffers` (hit) vs. read from disk (read). Aim for hits!
- **Rows:** Estimated vs. actual number of rows. Large discrepancies indicate outdated statistics.

## Tips for Analysis
1.  **Look for High Actual Times:** Focus on the nodes with the highest `actual time`.
2.  **Check for Disk Reads:** High `read` values in `BUFFERS` indicate I/O bottlenecks.
3.  **Check for Sequential Scans:** Large sequential scans on tables with > 10,000 rows usually need an index.
4.  **Implicit Conversions:** Look for `Filter: (column::text = 'val'::text)` which indicates a data type mismatch and prevents index usage.

## References
- [PostgreSQL Documentation: EXPLAIN](https://www.postgresql.org/docs/current/sql-explain.html)
- [PostgresQLExplain.com: Visualizer for JSON plans](https://explain.depesz.com/)
- [pganalyze: Explain Plan Guide](https://pganalyze.com/docs/explain)
