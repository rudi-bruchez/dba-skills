# Index Maintenance Scripts — Reference

## Fragmentation Analysis

### Check fragmentation (LIMITED mode — fast, suitable for all databases)

```sql
-- Use LIMITED mode for routine checks; SAMPLED for better accuracy on large databases
SELECT
    DB_NAME()                                   AS database_name,
    OBJECT_NAME(i.object_id)                    AS table_name,
    i.name                                      AS index_name,
    i.type_desc                                 AS index_type,
    ips.avg_fragmentation_in_percent,
    ips.page_count,
    ips.avg_page_space_used_in_percent,
    ips.record_count,
    ips.partition_number,
    -- Recommended action
    CASE
        WHEN ips.avg_fragmentation_in_percent > 30 THEN 'REBUILD'
        WHEN ips.avg_fragmentation_in_percent > 5  THEN 'REORGANIZE'
        ELSE 'NONE'
    END AS recommended_action
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED') ips
JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
WHERE ips.page_count > 100    -- Only indexes > 800 KB (smaller indexes tolerate fragmentation)
  AND ips.index_id > 0        -- Exclude heaps (index_id = 0)
ORDER BY ips.avg_fragmentation_in_percent DESC;
```

### Mode comparison

| Mode | Speed | Accuracy | When to use |
|---|---|---|---|
| `LIMITED` | Fastest | Allocation structures only | Routine maintenance scripts |
| `SAMPLED` | Moderate | ~1% of pages sampled | Weekly or monthly accurate reporting |
| `DETAILED` | Slowest | All pages read | One-time analysis; avoid during business hours on large tables |

---

## Rebuild and Reorganize

### Single index rebuild

```sql
-- Offline rebuild (blocks reads and writes; use during maintenance window)
ALTER INDEX IX_Orders_CustomerId ON dbo.Orders REBUILD;

-- Online rebuild (Enterprise Edition; minimal blocking)
ALTER INDEX IX_Orders_CustomerId ON dbo.Orders
REBUILD WITH (ONLINE = ON, FILLFACTOR = 80, SORT_IN_TEMPDB = ON, STATISTICS_NORECOMPUTE = OFF);

-- Rebuild all indexes on a table
ALTER INDEX ALL ON dbo.Orders REBUILD WITH (ONLINE = ON, FILLFACTOR = 80);

-- Rebuild a single partition
ALTER INDEX IX_FactSales_Date ON dbo.FactSales REBUILD PARTITION = 3;
```

### Reorganize

```sql
-- Always online and interruptible (safe during business hours)
ALTER INDEX IX_Orders_CustomerId ON dbo.Orders REORGANIZE;

-- Reorganize with LOB compaction (for tables with text/ntext/image/LOB columns)
ALTER INDEX IX_Docs_Content ON dbo.Documents REORGANIZE WITH (LOB_COMPACTION = ON);

-- Reorganize all indexes on a table
ALTER INDEX ALL ON dbo.Orders REORGANIZE;

-- IMPORTANT: REORGANIZE does NOT update statistics
UPDATE STATISTICS dbo.Orders WITH SAMPLE 30 PERCENT;
```

---

## Resumable Index Operations (SQL 2017+)

Allows pausing and resuming long-running online rebuilds to avoid maintenance window overruns.

```sql
-- Start resumable online rebuild
ALTER INDEX IX_Orders_CustomerId ON dbo.Orders
REBUILD WITH (ONLINE = ON, RESUMABLE = ON, MAX_DURATION = 120 MINUTES);
-- If 120 minutes elapses, the operation pauses automatically (not rolled back)

-- Check resumable operations in progress
SELECT
    name,
    state_desc,
    percent_complete,
    start_time,
    last_pause_time,
    total_execution_time
FROM sys.index_resumable_operations;

-- Pause the operation
ALTER INDEX IX_Orders_CustomerId ON dbo.Orders PAUSE;

-- Resume (optionally with new MAX_DURATION)
ALTER INDEX IX_Orders_CustomerId ON dbo.Orders RESUME WITH (MAX_DURATION = 60 MINUTES);

-- Abort (rolls back, no partial results kept)
ALTER INDEX IX_Orders_CustomerId ON dbo.Orders ABORT;
```

---

## Adaptive Maintenance Script

Automatically reorganizes 5–30% fragmented indexes and rebuilds > 30%.

```sql
DECLARE @TableName    NVARCHAR(300),
        @IndexName    NVARCHAR(300),
        @Fragmentation FLOAT,
        @SQL          NVARCHAR(600);

DECLARE idx_cursor CURSOR FAST_FORWARD FOR
    SELECT
        QUOTENAME(OBJECT_SCHEMA_NAME(i.object_id)) + '.' + QUOTENAME(OBJECT_NAME(i.object_id)),
        QUOTENAME(i.name),
        ips.avg_fragmentation_in_percent
    FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED') ips
    JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
    WHERE ips.page_count > 100
      AND ips.avg_fragmentation_in_percent > 5
      AND i.type > 0       -- Exclude heaps
      AND i.is_disabled = 0;

OPEN idx_cursor;
FETCH NEXT FROM idx_cursor INTO @TableName, @IndexName, @Fragmentation;

WHILE @@FETCH_STATUS = 0
BEGIN
    IF @Fragmentation > 30
        SET @SQL = 'ALTER INDEX ' + @IndexName + ' ON ' + @TableName
                 + ' REBUILD WITH (ONLINE = ON, SORT_IN_TEMPDB = ON, FILLFACTOR = 80)';
    ELSE
        SET @SQL = 'ALTER INDEX ' + @IndexName + ' ON ' + @TableName + ' REORGANIZE';

    PRINT @SQL;
    EXEC sp_executesql @SQL;

    FETCH NEXT FROM idx_cursor INTO @TableName, @IndexName, @Fragmentation;
END;

CLOSE idx_cursor;
DEALLOCATE idx_cursor;
```

**Note:** Consider Ola Hallengren's `IndexOptimize` as a production-grade alternative. Source: https://ola.hallengren.com — handles online/offline detection, partitions, columnstore, statistics, and scheduling.

---

## Columnstore Maintenance

```sql
-- Check columnstore rowgroup health
SELECT
    OBJECT_NAME(i.object_id)                    AS table_name,
    i.name                                      AS index_name,
    rg.state_desc                               AS rowgroup_state,
    COUNT(*)                                    AS rowgroup_count,
    SUM(rg.total_rows)                          AS total_rows,
    SUM(rg.deleted_rows)                        AS deleted_rows,
    CAST(SUM(rg.deleted_rows) * 100.0
         / NULLIF(SUM(rg.total_rows), 0) AS DECIMAL(5,2)) AS deleted_pct,
    SUM(rg.size_in_bytes) / 1048576.0           AS size_mb
FROM sys.column_store_row_groups rg
JOIN sys.indexes i ON rg.object_id = i.object_id AND rg.index_id = i.index_id
GROUP BY i.object_id, i.name, rg.state_desc
ORDER BY deleted_pct DESC;
-- Rebuild when deleted_pct > 10%

-- Rebuild columnstore (recompresses all rowgroups, removes deleted rows)
ALTER INDEX CCI_FactSales ON dbo.FactSales REBUILD;

-- Reorganize (compresses CLOSED rowgroups, promotes OPEN rows, removes TOMBSTONE)
ALTER INDEX CCI_FactSales ON dbo.FactSales REORGANIZE WITH (COMPRESS_ALL_ROW_GROUPS = ON);
```

---

## Heap Management

Heaps (tables with no clustered index) accumulate forwarding records from updates and are never automatically defragmented.

```sql
-- Find heaps with their sizes
SELECT
    t.name                                      AS table_name,
    SUM(p.rows)                                 AS row_count,
    SUM(a.total_pages) * 8 / 1024.0            AS total_mb,
    SUM(a.used_pages) * 8 / 1024.0             AS used_mb,
    (SUM(a.total_pages) - SUM(a.used_pages)) * 8 / 1024.0 AS free_mb
FROM sys.tables t
JOIN sys.indexes i ON t.object_id = i.object_id AND i.type = 0  -- 0 = heap
JOIN sys.partitions p ON i.object_id = p.object_id AND i.index_id = p.index_id
JOIN sys.allocation_units a ON p.partition_id = a.container_id
GROUP BY t.name
ORDER BY total_mb DESC;

-- Check forwarding record count (indicator of heap fragmentation)
SELECT
    OBJECT_NAME(ips.object_id)                  AS table_name,
    ips.forwarded_record_count,
    ips.page_count,
    ips.avg_page_space_used_in_percent
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'DETAILED') ips
WHERE ips.index_id = 0  -- Heaps only
  AND ips.forwarded_record_count > 0
ORDER BY ips.forwarded_record_count DESC;

-- Rebuild heap (removes forwarding records, reclaims space; brief table lock)
ALTER TABLE dbo.HeapTable REBUILD;
```

---

## Index Usage Statistics

```sql
-- Index usage since last SQL Server restart
SELECT
    OBJECT_NAME(ius.object_id)                  AS table_name,
    i.name                                      AS index_name,
    i.type_desc,
    ius.user_seeks,
    ius.user_scans,
    ius.user_lookups,
    ius.user_updates,
    ius.last_user_seek,
    ius.last_user_scan,
    ius.last_user_update,
    CASE WHEN (ius.user_seeks + ius.user_scans + ius.user_lookups) = 0
         THEN 'NEVER READ'
         ELSE 'READ'
    END AS read_status
FROM sys.dm_db_index_usage_stats ius
JOIN sys.indexes i ON ius.object_id = i.object_id AND ius.index_id = i.index_id
WHERE ius.database_id = DB_ID()
  AND i.type > 0
ORDER BY ius.user_updates DESC, (ius.user_seeks + ius.user_scans + ius.user_lookups) ASC;
```

---

## Index Operational Stats (Lock and Latch Waits)

```sql
-- Indexes with high lock or latch contention
SELECT
    OBJECT_NAME(ios.object_id)                  AS table_name,
    i.name                                      AS index_name,
    ios.row_lock_count,
    ios.row_lock_wait_count,
    ios.row_lock_wait_in_ms,
    ios.page_lock_count,
    ios.page_lock_wait_count,
    ios.page_lock_wait_in_ms,
    ios.page_latch_wait_count,
    ios.page_latch_wait_in_ms,
    ios.page_io_latch_wait_count,
    ios.page_io_latch_wait_in_ms
FROM sys.dm_db_index_operational_stats(DB_ID(), NULL, NULL, NULL) ios
JOIN sys.indexes i ON ios.object_id = i.object_id AND ios.index_id = i.index_id
WHERE ios.page_latch_wait_count > 1000
   OR ios.row_lock_wait_count > 1000
ORDER BY ios.page_latch_wait_in_ms + ios.row_lock_wait_in_ms DESC;
```

---

## Statistics Management

```sql
-- Update statistics with a 30% sample (good balance of speed and accuracy)
UPDATE STATISTICS dbo.Orders WITH SAMPLE 30 PERCENT;

-- Full scan for critical tables where estimates are still bad after sampled update
UPDATE STATISTICS dbo.Orders WITH FULLSCAN;

-- Check statistics update date for all indexes on a table
SELECT
    i.name                                      AS index_name,
    s.name                                      AS stats_name,
    sp.last_updated,
    sp.rows,
    sp.rows_sampled,
    CAST(sp.rows_sampled * 100.0 / NULLIF(sp.rows, 0) AS DECIMAL(5,2)) AS sample_pct,
    sp.modification_counter
FROM sys.indexes i
JOIN sys.stats s ON i.object_id = s.object_id AND i.index_id = s.stats_id
CROSS APPLY sys.dm_db_stats_properties(s.object_id, s.stats_id) sp
WHERE i.object_id = OBJECT_ID('dbo.Orders')
ORDER BY sp.last_updated DESC;
```
