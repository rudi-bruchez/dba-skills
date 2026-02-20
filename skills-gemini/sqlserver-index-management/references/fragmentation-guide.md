# Fragmentation Guide

Fragmentation occurs when index pages are out of physical order (external) or contain empty space (internal).

## 1. Types of Fragmentation
- **Internal Fragmentation:** Empty space on index pages (caused by low Fill Factor or page splits).
- **External Fragmentation:** Index pages are out of physical order (caused by inserts/updates).

## 2. Identifying Fragmentation
Use `sys.dm_db_index_physical_stats` to check the fragmentation level for all indexes.
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

## 3. Maintenance Thresholds
| Fragmentation (%) | Action |
|---|---|
| **< 5%** | No action required. |
| **5% - 30%** | **Reorganize** (`ALTER INDEX ... REORGANIZE`). |
| **> 30%** | **Rebuild** (`ALTER INDEX ... REBUILD`). |

## 4. Reorganize vs. Rebuild
- **Reorganize:** Online operation, low resource usage. Re-orders leaf-level pages and compacts data.
- **Rebuild:** Can be offline (standard edition) or online (enterprise edition). More resource-intensive. Re-creates the index from scratch.

## 5. Best Practices
- **Establish Baselines:** Regularly monitor fragmentation to understand "normal" levels for your environment.
- **Automated Alerts:** Set up SQL Agent alerts for high fragmentation or failed maintenance tasks.
- **Resumable Index Rebuilds:** Use `RESUMABLE = ON` for large index rebuilds in SQL Server 2017+ to allow pausing and resuming.

## References
- [Microsoft Documentation: Reorganize and Rebuild Indexes](https://learn.microsoft.com/en-us/sql/relational-databases/indexes/reorganize-and-rebuild-indexes)
- [Brent Ozar's Guide to Index Maintenance](https://www.brentozar.com/sql/index-maintenance/)
