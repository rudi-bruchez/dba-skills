# SQL Server Index Management Examples

Practical scenarios and how to implement index maintenance best practices in SQL Server.

## Scenario 1: Dealing with High Fragmentation

### Diagnosis
1.  **Monitor Fragmentation:** Use `sys.dm_db_index_physical_stats` to check the fragmentation level for all indexes.
2.  **Observation:** The `Sales` table's clustered index has 85% fragmentation.
```sql
SELECT
    OBJECT_NAME(ips.object_id) AS TableName,
    i.name AS IndexName,
    ips.avg_fragmentation_in_percent,
    ips.page_count
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'DETAILED') AS ips
JOIN sys.indexes AS i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
WHERE ips.avg_fragmentation_in_percent > 30
ORDER BY avg_fragmentation_in_percent DESC;
```
3.  **Action:** Rebuild the index (`ALTER INDEX ... REBUILD`).

### Procedure
```sql
ALTER INDEX [PK_Sales] ON [dbo].[Sales] REBUILD WITH (ONLINE = ON);
```
**Result:** Fragmentation is reduced to < 1%, significantly improving scan and lookup performance on the `Sales` table.

## Scenario 2: Identifying and Dropping Unused Indexes

### Diagnosis
1.  **Monitor Usage Stats:** Use `sys.dm_db_index_usage_stats` to identify indexes that are not being used but still consume storage and overhead.
2.  **Observation:** An index `idx_OldColumn` has no `idx_scan`, `idx_seek`, or `idx_lookup` but many `user_updates`.
```sql
SELECT
    OBJECT_NAME(s.object_id) AS TableName,
    i.name AS IndexName,
    s.user_seeks,
    s.user_scans,
    s.user_lookups,
    s.user_updates
FROM sys.dm_db_index_usage_stats AS s
JOIN sys.indexes AS i ON s.object_id = i.object_id AND s.index_id = i.index_id
WHERE s.database_id = DB_ID() AND i.index_id > 1 AND s.user_seeks = 0 AND s.user_scans = 0 AND s.user_lookups = 0;
```
3.  **Action:** Drop the unused index (`DROP INDEX ... ON ...`).

### Procedure
```sql
DROP INDEX [idx_OldColumn] ON [dbo].[Users];
```
**Result:** Storage is reclaimed, and write performance is improved by avoiding the overhead of updating the unused index during every `INSERT`, `UPDATE`, and `DELETE`.

## Scenario 3: Implementing a Filtered Index for Sparse Data

### Diagnosis
1.  **Issue:** A query is searching for "Pending" orders in a table with millions of rows, but only a small fraction are pending.
```sql
SELECT OrderID, CustomerID, OrderDate
FROM Sales.Orders
WHERE Status = 'Pending';
```
2.  **Observation:** A non-clustered index on `Status` is large and inefficient for this query.

### Optimization
Create a filtered index on `Status`.
```sql
CREATE NONCLUSTERED INDEX idx_pending_orders ON Sales.Orders (Status)
WHERE Status = 'Pending';
```
**Result:** The filtered index is much smaller and faster than a full non-clustered index, significantly improving performance for queries searching for pending orders.

## Scenario 4: Using Resumable Index Rebuilds for Large Tables

### Diagnosis
1.  **Goal:** Rebuild a large index on a table with billions of rows during a maintenance window, with the ability to pause and resume if necessary.

### Procedure
```sql
ALTER INDEX [PK_LargeTable] ON [dbo].[LargeTable] REBUILD WITH (RESUMABLE = ON, ONLINE = ON);

-- Pause the rebuild if the maintenance window is ending
ALTER INDEX [PK_LargeTable] ON [dbo].[LargeTable] PAUSE;

-- Resume the rebuild during the next maintenance window
ALTER INDEX [PK_LargeTable] ON [dbo].[LargeTable] RESUME;
```
**Result:** The large index rebuild is completed over multiple maintenance windows without having to start from scratch each time.
