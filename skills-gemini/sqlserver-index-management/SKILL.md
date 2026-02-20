---
name: sqlserver-index-management
description: Expert skill for managing SQL Server indexes, choosing the right types, and performing regular maintenance to optimize performance.
version: 1.0.0
tags:
  - sqlserver
  - dba
  - indexes
  - fragmentation
  - maintenance
---

# SQL Server Index Management

This skill provides expert guidance for choosing the right index types, monitoring for fragmentation, and performing regular maintenance tasks in SQL Server.

## Core Capabilities

- **Index Type Selection:** Choosing between clustered, non-clustered, unique, and filtered indexes.
- **Fragmentation Monitoring:** Identifying internal and external fragmentation using DMVs.
- **Index Maintenance:** Performing `REORGANIZE` and `REBUILD` operations based on fragmentation levels.
- **Missing & Unused Indexes:** Identifying indexes that the Optimizer recommends and those that are not being used.
- **Columnstore & Spatial Indexes:** Implementing specialized index types for analytical or geometric data.

## Workflow: Index Maintenance

1.  **Monitor Fragmentation:** Use `sys.dm_db_index_physical_stats` to check the fragmentation level for all indexes.
2.  **Determine Maintenance Action:**
    - **Reorganize (5% - 30%):** Online operation, low resource usage. Re-orders leaf-level pages and compacts data.
    - **Rebuild (> 30%):** Can be offline (standard edition) or online (enterprise edition). More resource-intensive. Re-creates the index from scratch.
3.  **Perform Maintenance:** Use `ALTER INDEX ... REORGANIZE` or `ALTER INDEX ... REBUILD`.
4.  **Identify Missing & Unused Indexes:** Use `sys.dm_db_missing_index_details` and `sys.dm_db_index_usage_stats`.
5.  **Clean Up Unused Indexes:** Drop indexes that have no `idx_scan` but still consume storage and overhead.

## Essential Commands

### 1. Monitor Fragmentation for All Indexes
```sql
SELECT
    OBJECT_NAME(ips.object_id) AS TableName,
    i.name AS IndexName,
    ips.avg_fragmentation_in_percent,
    ips.page_count
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'DETAILED') AS ips
JOIN sys.indexes AS i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
WHERE ips.avg_fragmentation_in_percent > 5
ORDER BY avg_fragmentation_in_percent DESC;
```

### 2. Reorganize an Index
```sql
ALTER INDEX [IndexName] ON [TableName] REORGANIZE;
```

### 3. Rebuild an Index (Online if Enterprise)
```sql
ALTER INDEX [IndexName] ON [TableName] REBUILD WITH (ONLINE = ON);
```

### 4. Identify Missing Indexes
```sql
SELECT TOP 10
    id.statement AS TableName,
    id.equality_columns,
    id.inequality_columns,
    id.included_columns,
    gs.avg_user_impact AS PotentialImpact
FROM sys.dm_db_missing_index_group_stats AS gs
JOIN sys.dm_db_missing_index_groups AS ig ON gs.group_handle = ig.index_group_handle
JOIN sys.dm_db_missing_index_details AS id ON ig.index_handle = id.index_handle
ORDER BY PotentialImpact DESC;
```

## Best Practices (2024-2025)

- **Use Ola Hallengren's Script:** The community standard for automated index and statistics maintenance.
- **Resumable Index Rebuilds:** Use `RESUMABLE = ON` for large index rebuilds in SQL Server 2017+ to allow pausing and resuming.
- **Monitor Fill Factor:** Use 80-90% for tables with frequent inserts to avoid page splits.
- **Automatic Index Management:** If using Azure SQL Database, leverage its automatic indexing features.
- **Avoid Over-Indexing:** Each index must be updated during every `INSERT`, `UPDATE`, and `DELETE`.

## References

- [Index Architecture Guide](./references/index-architecture-guide.md)
- [Fragmentation Guide](./references/fragmentation-guide.md)
- [Maintenance Scripts](./references/maintenance-scripts.md)
