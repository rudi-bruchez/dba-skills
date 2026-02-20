# Indexing Strategies

Indexes are critical for query performance, but too many can slow down writes. Choose the right type for each use case.

## Index Types

| Index Type | Description | Best Use Case |
|---|---|---|
| **Clustered Index** | Defines the physical order of data in the table. | Most tables should have a clustered index on a unique, non-changing column (e.g., `INT IDENTITY`). |
| **Non-Clustered Index** | A separate structure that points to the data rows. | For frequently searched or joined columns. |
| **Covering Index** | A non-clustered index that includes all columns requested by a query. | Eliminates "Key Lookups" by providing all necessary data from the index itself. |
| **Filtered Index** | A non-clustered index that includes only a subset of rows based on a `WHERE` clause. | For columns with sparse data (e.g., `WHERE IsProcessed = 0`). |
| **Unique Index** | Ensures that all values in the index are distinct. | For columns that must be unique (e.g., `EmailAddress`). |
| **Columnstore Index** | Optimized for analytical workloads (OLAP) and large data scans. | For tables with millions of rows being aggregated or analyzed. |

## Designing a Covering Index

If your query is:
```sql
SELECT OrderDate, CustomerID, TotalAmount
FROM Sales.Orders
WHERE Status = 'Pending';
```

A good covering index would be:
```sql
CREATE NONCLUSTERED INDEX IX_Orders_Status
ON Sales.Orders (Status)
INCLUDE (OrderDate, CustomerID, TotalAmount);
```

The `Status` column is in the index "key" for searching, and the `OrderDate`, `CustomerID`, and `TotalAmount` columns are in the `INCLUDE` clause to avoid lookups.

## Best Practices

1.  **Avoid Over-Indexing:** Each index must be updated during every `INSERT`, `UPDATE`, and `DELETE`.
2.  **Monitor Usage:** Use `sys.dm_db_index_usage_stats` to identify unused indexes that can be safely dropped.
3.  **Use 80-90% Fill Factor:** For tables with frequent updates to avoid page splits.
4.  **Keep Index Keys Narrow:** Avoid large `VARCHAR` columns in the index key. Use `INCLUDE` instead.
5.  **Maintain Fragmentation:** Regularly reorganize (5-30%) or rebuild (>30%) indexes to keep them efficient.

## References
- [Microsoft Documentation: Index Architecture and Design](https://learn.microsoft.com/en-us/sql/relational-databases/sql-server-index-design-guide)
- [Ola Hallengren's Index Maintenance Script](https://ola.hallengren.com/sql-server-index-and-statistics-maintenance.html)
