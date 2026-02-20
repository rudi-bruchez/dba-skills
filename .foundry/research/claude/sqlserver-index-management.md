# SQL Server Index Management

## Overview

Indexes are the primary tool for query performance in SQL Server. Proper index design, maintenance, and monitoring are critical DBA responsibilities. Over-indexing harms write performance; under-indexing harms read performance. Index maintenance (rebuild/reorganize) addresses fragmentation.

---

## Index Types

### Clustered Index
- Defines the physical storage order of table rows.
- One per table (or zero — "heap").
- Row data is stored in the leaf level of the B-tree.
- Best on: primary key (auto-created), range query columns, frequently joined columns.
- Avoid on: frequently updated columns (causes row movement), GUIDs (random inserts cause fragmentation).

### Non-Clustered Index
- Separate B-tree structure with pointers back to the clustered index or heap.
- Up to 999 per table (practical limit much lower; target < 10-15 per OLTP table).
- Leaf level contains index key + row locator (RID or clustered index key).
- Add INCLUDE columns to make covering indexes that avoid key lookups.

### Columnstore Index (SQL 2012+)
- Column-oriented storage with high compression ratios (10:1 typical).
- Optimized for analytics / aggregation queries (reads millions of rows fast).
- Clustered Columnstore Index: entire table stored as columnstore. Best for DW/analytics.
- Non-Clustered Columnstore: OLTP table with a columnstore for analytical queries. "Dual" access.
- Uses batch mode processing for high throughput aggregations.

### Filtered Index
- Non-clustered index with a WHERE clause (partial index).
- Smaller and more efficient than full index when queries always filter on a specific value.
- Statistics maintained only for filtered rows.

### Unique Index
- Enforces uniqueness constraint. Can be clustered or non-clustered.
- Created automatically by PRIMARY KEY and UNIQUE constraints.

### Full-Text Index
- Enables linguistic search, CONTAINS, FREETEXT queries.
- Separate from B-tree index; managed by Full-Text Engine.
- Requires Full-Text Catalog and special maintenance.

### XML Index
- Primary XML index: shreds XML values for efficient querying.
- Secondary XML indexes: PATH, VALUE, PROPERTY for different query patterns.

### Spatial Index
- For geography and geometry data types.

### Hash Index (In-Memory OLTP / Hekaton)
- Only for memory-optimized tables.
- Equality lookups only; no range scans.
- Requires specifying bucket count at creation.

---

## Index Creation Syntax

### Basic Non-Clustered Index
```sql
CREATE NONCLUSTERED INDEX IX_Orders_CustomerId_OrderDate
ON dbo.Orders (customer_id ASC, order_date DESC)
INCLUDE (total_amount, status)  -- Covering columns (not part of key)
WHERE status IN ('Pending', 'Processing')  -- Filtered index
WITH (
    FILLFACTOR = 80,            -- Leave 20% free for inserts
    PAD_INDEX = ON,             -- Apply fillfactor to intermediate pages
    SORT_IN_TEMPDB = ON,        -- Build sort in tempdb (faster)
    ONLINE = ON,                -- Non-blocking rebuild (Enterprise edition)
    DATA_COMPRESSION = PAGE     -- Page compression
)
ON [PRIMARY];
```

### Clustered Index
```sql
CREATE CLUSTERED INDEX CIX_Orders_OrderId
ON dbo.Orders (order_id ASC)
WITH (FILLFACTOR = 95, SORT_IN_TEMPDB = ON, ONLINE = ON);
```

### Clustered Columnstore Index
```sql
CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactSales
ON dbo.FactSales
WITH (DATA_COMPRESSION = COLUMNSTORE_ARCHIVE, ONLINE = ON);
-- COLUMNSTORE = default compression, COLUMNSTORE_ARCHIVE = higher ratio, slower access
```

### Non-Clustered Columnstore
```sql
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_Orders_Analytics
ON dbo.Orders (order_date, customer_id, product_id, quantity, unit_price, total_amount)
WHERE status = 'Completed';  -- Filtered (SQL 2016+)
```

---

## Fragmentation

### What Causes Fragmentation
- **Logical fragmentation**: Pages out of order in the index B-tree. Caused by page splits from inserts/updates.
- **Page splits**: When a data page is full, SQL splits it — creating two ~50% full pages. Low FILLFACTOR triggers splits faster.
- **External fragmentation**: Pages not contiguous on disk (less impactful with SSD).
- **Internal fragmentation**: Pages with much free space (caused by deletes or low FILLFACTOR).

### Fragmentation Thresholds
| Fragmentation Level | Action | Method |
|---|---|---|
| 0 - 5% | No action needed | None |
| 5 - 30% | REORGANIZE | Online, incremental, interruptable |
| > 30% | REBUILD | Offline (or ONLINE on Enterprise) |
| Any (columnstore) | REBUILD to remove deleted rows | Check `deleted_rows > total_rows * 0.1` |

### Check Fragmentation
```sql
-- Check fragmentation for a specific database
SELECT
    DB_NAME() AS database_name,
    OBJECT_NAME(i.object_id) AS table_name,
    i.name AS index_name,
    i.type_desc AS index_type,
    ips.avg_fragmentation_in_percent,
    ips.page_count,
    ips.avg_page_space_used_in_percent,
    ips.record_count,
    ips.partition_number
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED') ips
JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
WHERE ips.page_count > 100  -- Only meaningful indexes (> 100 pages = 800KB)
  AND ips.index_id > 0      -- Exclude heaps (index_id = 0 means heap)
ORDER BY ips.avg_fragmentation_in_percent DESC;
```

### SAMPLED vs LIMITED vs DETAILED Mode
- `LIMITED`: Fastest, reads only allocation structures. Sufficient for most maintenance.
- `SAMPLED`: Reads ~1% of pages for better accuracy. Balance of speed and accuracy.
- `DETAILED`: Reads all pages. Most accurate but slowest; avoid on large tables during business hours.

---

## Index Maintenance

### Rebuild (Full Reconstruction)
```sql
-- Rebuild specific index (offline by default)
ALTER INDEX IX_Orders_CustomerId ON dbo.Orders REBUILD;

-- Rebuild with options
ALTER INDEX IX_Orders_CustomerId ON dbo.Orders
REBUILD WITH (FILLFACTOR = 80, SORT_IN_TEMPDB = ON, STATISTICS_NORECOMPUTE = OFF, ONLINE = ON);

-- Rebuild all indexes on a table
ALTER INDEX ALL ON dbo.Orders REBUILD WITH (ONLINE = ON, FILLFACTOR = 80);

-- Rebuild with partition
ALTER INDEX IX_FactSales_Date ON dbo.FactSales
REBUILD PARTITION = 3;
```

### Reorganize (Incremental Defragmentation)
```sql
-- Reorganize specific index (always online, interruptable)
ALTER INDEX IX_Orders_CustomerId ON dbo.Orders REORGANIZE;

-- Reorganize with LOB compaction
ALTER INDEX IX_Docs_Content ON dbo.Documents REORGANIZE WITH (LOB_COMPACTION = ON);

-- Reorganize all indexes on a table
ALTER INDEX ALL ON dbo.Orders REORGANIZE;
```

### Statistics Update After Reorganize
```sql
-- REORGANIZE does NOT update statistics; do it manually
UPDATE STATISTICS dbo.Orders WITH SAMPLE 30 PERCENT;
-- REBUILD does update statistics as part of the operation
```

### Resumable Index Operations (SQL 2017+ for online, SQL 2019+ for offline)
```sql
-- Start resumable rebuild
ALTER INDEX IX_Orders_CustomerId ON dbo.Orders
REBUILD WITH (ONLINE = ON, RESUMABLE = ON, MAX_DURATION = 120 MINUTES);

-- Pause it
ALTER INDEX IX_Orders_CustomerId ON dbo.Orders PAUSE;

-- Resume it
ALTER INDEX IX_Orders_CustomerId ON dbo.Orders RESUME WITH (MAX_DURATION = 60 MINUTES);

-- Abort
ALTER INDEX IX_Orders_CustomerId ON dbo.Orders ABORT;

-- Check resumable operations
SELECT * FROM sys.index_resumable_operations;
```

---

## Index Usage Statistics

### Index Usage (seeks, scans, lookups, updates)
```sql
-- Index usage statistics since last restart
SELECT
    DB_NAME(ius.database_id) AS database_name,
    OBJECT_NAME(ius.object_id) AS table_name,
    i.name AS index_name,
    i.type_desc AS index_type,
    ius.user_seeks,
    ius.user_scans,
    ius.user_lookups,
    ius.user_updates,
    ius.last_user_seek,
    ius.last_user_scan,
    ius.last_user_update,
    -- Maintenance cost ratio: updates >> reads = potentially unused index
    CASE WHEN (ius.user_seeks + ius.user_scans + ius.user_lookups) = 0
         THEN 'NEVER READ'
         ELSE 'READ'
    END AS read_status
FROM sys.dm_db_index_usage_stats ius
JOIN sys.indexes i ON ius.object_id = i.object_id AND ius.index_id = i.index_id
WHERE ius.database_id = DB_ID()
  AND i.type > 0  -- Exclude heaps
ORDER BY ius.user_updates DESC, (ius.user_seeks + ius.user_scans + ius.user_lookups) ASC;
```

### Unused Indexes (candidates for dropping)
```sql
-- Indexes never read since last restart (high write cost, zero read benefit)
SELECT
    OBJECT_NAME(i.object_id) AS table_name,
    i.name AS index_name,
    i.type_desc,
    ISNULL(ius.user_seeks, 0) AS user_seeks,
    ISNULL(ius.user_scans, 0) AS user_scans,
    ISNULL(ius.user_lookups, 0) AS user_lookups,
    ISNULL(ius.user_updates, 0) AS user_updates,
    'DROP INDEX ' + QUOTENAME(i.name) + ' ON ' + QUOTENAME(OBJECT_NAME(i.object_id)) AS drop_statement
FROM sys.indexes i
LEFT JOIN sys.dm_db_index_usage_stats ius
    ON i.object_id = ius.object_id AND i.index_id = ius.index_id AND ius.database_id = DB_ID()
WHERE i.type > 0  -- Exclude heaps and clustered (never drop clustered without analysis)
  AND i.is_unique = 0  -- Don't drop unique indexes (may enforce constraints)
  AND i.is_primary_key = 0
  AND OBJECT_NAME(i.object_id) IS NOT NULL
  AND ISNULL(ius.user_seeks, 0) = 0
  AND ISNULL(ius.user_scans, 0) = 0
  AND ISNULL(ius.user_lookups, 0) = 0
ORDER BY ISNULL(ius.user_updates, 0) DESC;
```

### Duplicate / Redundant Indexes
```sql
-- Find indexes with identical or overlapping key columns
SELECT
    t1.name AS table_name,
    i1.name AS index1,
    i2.name AS index2,
    STUFF((SELECT ', ' + COL_NAME(ic1.object_id, ic1.column_id)
           FROM sys.index_columns ic1
           WHERE ic1.object_id = i1.object_id AND ic1.index_id = i1.index_id
           ORDER BY ic1.key_ordinal
           FOR XML PATH('')), 1, 2, '') AS index1_keys,
    STUFF((SELECT ', ' + COL_NAME(ic2.object_id, ic2.column_id)
           FROM sys.index_columns ic2
           WHERE ic2.object_id = i2.object_id AND ic2.index_id = i2.index_id
           ORDER BY ic2.key_ordinal
           FOR XML PATH('')), 1, 2, '') AS index2_keys
FROM sys.indexes i1
JOIN sys.indexes i2 ON i1.object_id = i2.object_id
    AND i1.index_id < i2.index_id
JOIN sys.tables t1 ON i1.object_id = t1.object_id
WHERE i1.type > 0 AND i2.type > 0
  AND NOT EXISTS (  -- Same leading key column
      SELECT 1 FROM sys.index_columns ic1
      JOIN sys.index_columns ic2
          ON ic1.column_id = ic2.column_id
          AND ic2.object_id = i2.object_id
          AND ic2.index_id = i2.index_id
          AND ic2.key_ordinal = 1
      WHERE ic1.object_id = i1.object_id
        AND ic1.index_id = i1.index_id
        AND ic1.key_ordinal = 1
        AND ic1.column_id != ic2.column_id
  );
```

---

## Missing Index Analysis

```sql
-- Top missing indexes with estimated improvement
SELECT TOP 20
    DB_NAME(mid.database_id) AS database_name,
    OBJECT_NAME(mid.object_id, mid.database_id) AS table_name,
    migs.user_seeks,
    migs.user_scans,
    ROUND(migs.avg_total_user_cost * migs.avg_user_impact * (migs.user_seeks + migs.user_scans), 0) AS impact,
    migs.avg_user_impact AS avg_pct_improvement,
    mid.equality_columns,
    mid.inequality_columns,
    mid.included_columns,
    migs.last_user_seek,
    -- Suggested CREATE INDEX statement
    'CREATE NONCLUSTERED INDEX IX_' +
        REPLACE(REPLACE(OBJECT_NAME(mid.object_id, mid.database_id), ' ', '_'), '.', '_') + '_' +
        CONVERT(VARCHAR(10), mig.index_group_handle) +
        ' ON ' + mid.statement +
        ' (' + ISNULL(mid.equality_columns, '') +
        CASE WHEN mid.inequality_columns IS NOT NULL THEN
            CASE WHEN mid.equality_columns IS NOT NULL THEN ', ' ELSE '' END + mid.inequality_columns
        ELSE '' END + ')' +
        ISNULL(' INCLUDE (' + mid.included_columns + ')', '') AS suggested_index
FROM sys.dm_db_missing_index_details mid
JOIN sys.dm_db_missing_index_groups mig ON mid.index_handle = mig.index_handle
JOIN sys.dm_db_missing_index_group_stats migs ON mig.index_group_handle = migs.group_handle
WHERE mid.database_id = DB_ID()
ORDER BY impact DESC;
```

---

## Index Operational Stats (Wait Times per Index)

```sql
-- Index operational statistics including locking, latching, row locks
SELECT
    OBJECT_NAME(ios.object_id) AS table_name,
    i.name AS index_name,
    ios.leaf_insert_count + ios.leaf_delete_count + ios.leaf_update_count AS leaf_dml_ops,
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
WHERE ios.page_latch_wait_count > 0
ORDER BY ios.page_latch_wait_in_ms DESC;
```

---

## FILLFACTOR Guidelines

| Scenario | Suggested FILLFACTOR |
|----------|---------------------|
| Static / read-only table | 100% (no free space) |
| Mostly read, rare inserts | 90-95% |
| Balanced read/write OLTP | 80-85% |
| Sequential insert (identity PK) | 100% (inserts always at end) |
| GUID/random PK (fragmentation-prone) | 70-80% |
| Heavily updated volatile table | 70-75% |
| Reporting / DW | 100% |

```sql
-- Set instance-level default fillfactor
EXEC sp_configure 'fill factor (%)', 80;
RECONFIGURE;

-- Check current fillfactor on indexes
SELECT OBJECT_NAME(object_id) AS table_name, name AS index_name, fill_factor
FROM sys.indexes
WHERE fill_factor > 0 AND fill_factor < 100
ORDER BY OBJECT_NAME(object_id), name;
```

---

## Heap Management

```sql
-- Find heaps (tables with no clustered index)
SELECT
    t.name AS table_name,
    SUM(p.rows) AS row_count,
    SUM(a.total_pages) * 8 / 1024.0 AS total_mb,
    SUM(a.used_pages) * 8 / 1024.0 AS used_mb
FROM sys.tables t
JOIN sys.indexes i ON t.object_id = i.object_id AND i.type = 0  -- 0 = heap
JOIN sys.partitions p ON i.object_id = p.object_id AND i.index_id = p.index_id
JOIN sys.allocation_units a ON p.partition_id = a.container_id
GROUP BY t.name
ORDER BY total_mb DESC;

-- Rebuild a heap (removes forwarding records, reclaims space)
ALTER TABLE dbo.HeapTable REBUILD;
```

---

## Index Maintenance Script (Adaptive)

```sql
-- Adaptive maintenance: reorganize 5-30%, rebuild > 30%
DECLARE @TableName NVARCHAR(200), @IndexName NVARCHAR(200), @Fragmentation FLOAT, @SQL NVARCHAR(500);

DECLARE idx_cursor CURSOR FOR
SELECT
    QUOTENAME(OBJECT_SCHEMA_NAME(i.object_id)) + '.' + QUOTENAME(OBJECT_NAME(i.object_id)),
    QUOTENAME(i.name),
    ips.avg_fragmentation_in_percent
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED') ips
JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
WHERE ips.page_count > 100
  AND ips.avg_fragmentation_in_percent > 5
  AND i.type > 0;

OPEN idx_cursor;
FETCH NEXT FROM idx_cursor INTO @TableName, @IndexName, @Fragmentation;
WHILE @@FETCH_STATUS = 0
BEGIN
    IF @Fragmentation > 30
        SET @SQL = 'ALTER INDEX ' + @IndexName + ' ON ' + @TableName + ' REBUILD WITH (ONLINE = ON, SORT_IN_TEMPDB = ON)';
    ELSE
        SET @SQL = 'ALTER INDEX ' + @IndexName + ' ON ' + @TableName + ' REORGANIZE';

    PRINT @SQL;
    -- EXEC sp_executesql @SQL;  -- Uncomment to execute
    FETCH NEXT FROM idx_cursor INTO @TableName, @IndexName, @Fragmentation;
END;
CLOSE idx_cursor;
DEALLOCATE idx_cursor;
```

---

## Columnstore-Specific Maintenance

```sql
-- Check columnstore health (deleted rows, rowgroup states)
SELECT
    OBJECT_NAME(i.object_id) AS table_name,
    i.name AS index_name,
    rg.state_desc AS rowgroup_state,
    COUNT(*) AS rowgroup_count,
    SUM(rg.total_rows) AS total_rows,
    SUM(rg.deleted_rows) AS deleted_rows,
    SUM(rg.size_in_bytes) / 1024.0 / 1024.0 AS size_mb
FROM sys.column_store_row_groups rg
JOIN sys.indexes i ON rg.object_id = i.object_id AND rg.index_id = i.index_id
GROUP BY i.object_id, i.name, rg.state_desc
ORDER BY i.object_id, rg.state_desc;

-- Rowgroup states:
-- OPEN:        Delta store (row mode), not yet compressed
-- CLOSED:      Ready for compression (Tuple Mover will process)
-- COMPRESSED:  Compressed columnstore rowgroup (target: ~1,048,576 rows per rowgroup)
-- TOMBSTONE:   Being deleted
-- INVISIBLE:   Being replaced

-- Rebuild columnstore when deleted_rows > 10% of total_rows
ALTER INDEX CCI_FactSales ON dbo.FactSales REBUILD;

-- Reorganize (compresses CLOSED rowgroups, removes TOMBSTONE)
ALTER INDEX CCI_FactSales ON dbo.FactSales REORGANIZE WITH (COMPRESS_ALL_ROW_GROUPS = ON);
```

---

## Key Metrics and Thresholds

| Metric | Source | Threshold |
|--------|--------|-----------|
| Index fragmentation | sys.dm_db_index_physical_stats | Reorganize > 5%; Rebuild > 30% |
| Unused indexes (reads = 0) | sys.dm_db_index_usage_stats | Review and drop candidates |
| Missing index impact score | sys.dm_db_missing_index_group_stats | Implement if impact > 100,000 |
| Key lookups per sec | sys.dm_db_index_usage_stats | High lookups = add INCLUDE columns |
| Average index key size | sys.indexes | Keep composite key < 900 bytes |
| Indexes per OLTP table | sys.indexes | Target < 10-15 |
| Columnstore deleted rows | sys.column_store_row_groups | Rebuild if > 10% deleted |
| Forwarded fetch count (heap) | sys.dm_db_index_physical_stats | Rebuild heap if > 5% of rows |
