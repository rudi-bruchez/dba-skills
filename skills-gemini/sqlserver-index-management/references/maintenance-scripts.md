# Maintenance Scripts

A collection of essential T-SQL scripts for automated index and statistics maintenance.

## 1. Ola Hallengren's Solution
The community standard for automated index and statistics maintenance in SQL Server.
- **Website:** [https://ola.hallengren.com/](https://ola.hallengren.com/)
- **Installation:** Download the `MaintenanceSolution.sql` script and execute it in your `master` or a separate maintenance database.
- **Example Command:**
```sql
EXECUTE dbo.IndexOptimize
@Databases = 'USER_DATABASES',
@FragmentationLow = NULL,
@FragmentationMedium = 'INDEX_REORGANIZE,INDEX_REBUILD_ONLINE,INDEX_REBUILD_OFFLINE',
@FragmentationHigh = 'INDEX_REBUILD_ONLINE,INDEX_REBUILD_OFFLINE',
@FragmentationLevel1 = 5,
@FragmentationLevel2 = 30,
@UpdateStatistics = 'ALL',
@OnlyModifiedStatistics = 'Y';
```

## 2. Minion Maintenance (Alternative)
Another popular and powerful community solution for index maintenance.
- **Website:** [http://minionware.net/maintenance/](http://minionware.net/maintenance/)

## 3. Resumable Index Rebuilds (SQL 2017+)
For large indexes that take a long time to rebuild, use resumable operations to allow pausing and resuming.
```sql
ALTER INDEX [IndexName] ON [TableName] REBUILD WITH (RESUMABLE = ON, ONLINE = ON);

-- Pause the rebuild
ALTER INDEX [IndexName] ON [TableName] PAUSE;

-- Resume the rebuild
ALTER INDEX [IndexName] ON [TableName] RESUME;

-- Abort the rebuild
ALTER INDEX [IndexName] ON [TableName] ABORT;
```

## 4. Identifying Unused Indexes
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
WHERE s.database_id = DB_ID() AND OBJECTPROPERTY(s.object_id, 'IsUserTable') = 1
  AND i.index_id > 1 AND s.user_seeks = 0 AND s.user_scans = 0 AND s.user_lookups = 0
ORDER BY s.user_updates DESC;
```

## Best Practices
- **Schedule Maintenance:** Perform regular maintenance tasks during off-peak hours to minimize impact on the workload.
- **Log Activity:** Maintain a log of all index maintenance operations for auditing and troubleshooting.
- **Verify Success:** Regularly check the maintenance logs to ensure that tasks are completing successfully.

## References
- [Ola Hallengren's Index Maintenance Script](https://ola.hallengren.com/sql-server-index-and-statistics-maintenance.html)
- [Microsoft Documentation: Resumable Index Operations](https://learn.microsoft.com/en-us/sql/relational-databases/indexes/resumable-index-operations)
