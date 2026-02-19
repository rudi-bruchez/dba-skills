---
name: sqlserver-index-management
description: Manages SQL Server indexes including fragmentation analysis, missing index evaluation, unused index detection, and maintenance strategy design. Use when diagnosing slow queries due to missing indexes, planning index maintenance windows, auditing index effectiveness, or designing indexes for new tables.
metadata:
  author: dba-skills
  version: "1.0"
  platform: SQL Server 2016+
---

# SQL Server Index Management

## Index Health Overview

Run this query first to get a summary of index health in the current database.

```sql
-- Summary: fragmentation, usage, and size for all non-clustered indexes
SELECT
    OBJECT_NAME(i.object_id)          AS table_name,
    i.name                            AS index_name,
    i.type_desc                       AS index_type,
    ips.avg_fragmentation_in_percent  AS fragmentation_pct,
    ips.page_count,
    ISNULL(ius.user_seeks, 0)         AS user_seeks,
    ISNULL(ius.user_scans, 0)         AS user_scans,
    ISNULL(ius.user_lookups, 0)       AS user_lookups,
    ISNULL(ius.user_updates, 0)       AS user_updates,
    ius.last_user_seek
FROM sys.indexes i
JOIN sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED') ips
    ON i.object_id = ips.object_id AND i.index_id = ips.index_id
LEFT JOIN sys.dm_db_index_usage_stats ius
    ON i.object_id = ius.object_id AND i.index_id = ius.index_id
    AND ius.database_id = DB_ID()
WHERE ips.page_count > 100   -- Ignore tiny indexes (< 800 KB)
  AND i.type > 0             -- Exclude heaps
ORDER BY ips.avg_fragmentation_in_percent DESC;
```

---

## Missing Index Analysis Workflow

The SQL Server optimizer records missed index opportunities in DMVs. Use this workflow to evaluate them.

### Step 1 — Get top missing index recommendations with impact score
```sql
SELECT TOP 20
    DB_NAME(mid.database_id)                          AS database_name,
    OBJECT_NAME(mid.object_id, mid.database_id)       AS table_name,
    migs.user_seeks,
    migs.user_scans,
    ROUND(migs.avg_total_user_cost * migs.avg_user_impact
          * (migs.user_seeks + migs.user_scans), 0)   AS impact_score,
    migs.avg_user_impact                              AS avg_pct_improvement,
    mid.equality_columns,
    mid.inequality_columns,
    mid.included_columns,
    migs.last_user_seek,
    'CREATE NONCLUSTERED INDEX IX_'
        + REPLACE(REPLACE(OBJECT_NAME(mid.object_id, mid.database_id), ' ', '_'), '.', '_')
        + '_' + CONVERT(VARCHAR(10), mig.index_group_handle)
        + ' ON ' + mid.statement
        + ' (' + ISNULL(mid.equality_columns, '')
        + CASE WHEN mid.inequality_columns IS NOT NULL
               THEN CASE WHEN mid.equality_columns IS NOT NULL THEN ', ' ELSE '' END
                    + mid.inequality_columns
          ELSE '' END + ')'
        + ISNULL(' INCLUDE (' + mid.included_columns + ')', '')
        AS suggested_index
FROM sys.dm_db_missing_index_details mid
JOIN sys.dm_db_missing_index_groups mig
    ON mid.index_handle = mig.index_handle
JOIN sys.dm_db_missing_index_group_stats migs
    ON mig.index_group_handle = migs.group_handle
WHERE mid.database_id = DB_ID()
ORDER BY impact_score DESC;
```

### Step 2 — Evaluation checklist before creating an index

- Impact score > 100,000? (Lower scores rarely justify the write overhead.)
- Does an existing index already cover the equality columns? If so, add INCLUDE columns to it instead.
- Are equality columns listed before inequality columns in the suggested statement? (The DMV always does this correctly — verify before copying.)
- Will this index be maintained during write-heavy operations? Consider FILLFACTOR.
- How many indexes does this table already have? OLTP tables: target < 10–15 non-clustered indexes.

---

## Fragmentation Assessment

### Check fragmentation (LIMITED mode for speed)
```sql
SELECT
    DB_NAME()                          AS database_name,
    OBJECT_NAME(i.object_id)           AS table_name,
    i.name                             AS index_name,
    i.type_desc                        AS index_type,
    ips.avg_fragmentation_in_percent,
    ips.page_count,
    ips.avg_page_space_used_in_percent,
    ips.record_count,
    ips.partition_number
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED') ips
JOIN sys.indexes i
    ON ips.object_id = i.object_id AND ips.index_id = i.index_id
WHERE ips.page_count > 100
  AND ips.index_id > 0
ORDER BY ips.avg_fragmentation_in_percent DESC;
```

**Fragmentation thresholds and recommended action:**

| Fragmentation | Action | Notes |
|---|---|---|
| 0 – 5% | No action | Below the threshold of meaningful impact |
| 5 – 30% | REORGANIZE | Online, interruptible, incremental; always safe |
| > 30% | REBUILD | Standard: offline. Enterprise: add `ONLINE = ON` |
| Columnstore (any) | REBUILD when `deleted_rows > 10%` of `total_rows` | Use `sys.column_store_row_groups` to check |

### Rebuild an index
```sql
-- Standard rebuild (offline — blocks reads and writes)
ALTER INDEX IX_Orders_CustomerId ON dbo.Orders REBUILD;

-- Online rebuild (Enterprise Edition only — minimal blocking)
ALTER INDEX IX_Orders_CustomerId ON dbo.Orders
REBUILD WITH (ONLINE = ON, FILLFACTOR = 80, SORT_IN_TEMPDB = ON);

-- Rebuild all indexes on a table
ALTER INDEX ALL ON dbo.Orders REBUILD WITH (ONLINE = ON, FILLFACTOR = 80);
```

### Reorganize an index
```sql
-- Always online and interruptible
ALTER INDEX IX_Orders_CustomerId ON dbo.Orders REORGANIZE;

-- REORGANIZE does NOT update statistics — update manually afterward
UPDATE STATISTICS dbo.Orders WITH SAMPLE 30 PERCENT;
```

---

## Unused Index Detection

Unused indexes consume write performance and disk space with no read benefit. Review periodically — note that DMVs reset on service restart.

```sql
-- Indexes never read since last SQL Server restart (high write cost, zero read benefit)
SELECT
    OBJECT_NAME(i.object_id)  AS table_name,
    i.name                    AS index_name,
    i.type_desc,
    ISNULL(ius.user_seeks, 0)   AS user_seeks,
    ISNULL(ius.user_scans, 0)   AS user_scans,
    ISNULL(ius.user_lookups, 0) AS user_lookups,
    ISNULL(ius.user_updates, 0) AS user_updates,
    ius.last_user_seek,
    ius.last_user_scan,
    'DROP INDEX ' + QUOTENAME(i.name) + ' ON '
        + QUOTENAME(OBJECT_SCHEMA_NAME(i.object_id)) + '.'
        + QUOTENAME(OBJECT_NAME(i.object_id)) AS drop_statement
FROM sys.indexes i
LEFT JOIN sys.dm_db_index_usage_stats ius
    ON i.object_id = ius.object_id AND i.index_id = ius.index_id
    AND ius.database_id = DB_ID()
WHERE i.type > 0
  AND i.is_unique = 0
  AND i.is_primary_key = 0
  AND OBJECT_NAME(i.object_id) IS NOT NULL
  AND ISNULL(ius.user_seeks, 0) = 0
  AND ISNULL(ius.user_scans, 0) = 0
  AND ISNULL(ius.user_lookups, 0) = 0
ORDER BY ISNULL(ius.user_updates, 0) DESC;
-- Never drop unique indexes or primary keys without analysis
-- Verify the server has been running long enough to capture all workloads (> 1 week ideally)
```

---

## Index Maintenance Workflow

### Adaptive maintenance script (5–30% reorganize, > 30% rebuild)
```sql
DECLARE @TableName NVARCHAR(300), @IndexName NVARCHAR(300),
        @Fragmentation FLOAT, @SQL NVARCHAR(600);

DECLARE idx_cursor CURSOR FOR
    SELECT
        QUOTENAME(OBJECT_SCHEMA_NAME(i.object_id)) + '.'
            + QUOTENAME(OBJECT_NAME(i.object_id)),
        QUOTENAME(i.name),
        ips.avg_fragmentation_in_percent
    FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED') ips
    JOIN sys.indexes i
        ON ips.object_id = i.object_id AND ips.index_id = i.index_id
    WHERE ips.page_count > 100
      AND ips.avg_fragmentation_in_percent > 5
      AND i.type > 0;

OPEN idx_cursor;
FETCH NEXT FROM idx_cursor INTO @TableName, @IndexName, @Fragmentation;
WHILE @@FETCH_STATUS = 0
BEGIN
    IF @Fragmentation > 30
        SET @SQL = 'ALTER INDEX ' + @IndexName + ' ON ' + @TableName
                 + ' REBUILD WITH (ONLINE = ON, SORT_IN_TEMPDB = ON)';
    ELSE
        SET @SQL = 'ALTER INDEX ' + @IndexName + ' ON ' + @TableName + ' REORGANIZE';

    EXEC sp_executesql @SQL;
    FETCH NEXT FROM idx_cursor INTO @TableName, @IndexName, @Fragmentation;
END;
CLOSE idx_cursor;
DEALLOCATE idx_cursor;
```

**Industry-standard alternative:** Ola Hallengren's `IndexOptimize` stored procedure. Source: https://ola.hallengren.com

---

## Index Design Principles

### Covering index pattern (eliminate key lookups)
```sql
-- Query: SELECT name, email FROM dbo.Customers WHERE country = 'US' AND status = 'Active'
CREATE NONCLUSTERED INDEX IX_Customers_Country_Status
ON dbo.Customers (country, status)   -- Seek predicates in key
INCLUDE (name, email);               -- Projected columns as included (not in key)
```

### Filtered index (partial index for sparse queries)
```sql
-- Only index pending orders; smaller, faster for this specific query pattern
CREATE NONCLUSTERED INDEX IX_Orders_Pending
ON dbo.Orders (customer_id, order_date)
INCLUDE (total_amount)
WHERE status = 'Pending';
```

### Columnstore for analytical queries on OLTP tables
```sql
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_Sales_Analytics
ON dbo.Sales (sale_date, product_id, customer_id, amount, quantity);
-- Batch mode processing; 10:1 typical compression
```

**FILLFACTOR guidelines:**

| Scenario | FILLFACTOR |
|---|---|
| Sequential identity PK | 100 (inserts always at end) |
| Balanced OLTP read/write | 80–85 |
| Heavily updated / GUID PK | 70–80 |
| Read-only or static table | 100 |
| Reporting / data warehouse | 100 |

---

## Columnstore Index Health Check

```sql
SELECT
    OBJECT_NAME(i.object_id)  AS table_name,
    i.name                    AS index_name,
    rg.state_desc             AS rowgroup_state,
    COUNT(*)                  AS rowgroup_count,
    SUM(rg.total_rows)        AS total_rows,
    SUM(rg.deleted_rows)      AS deleted_rows,
    CAST(SUM(rg.deleted_rows) * 100.0
         / NULLIF(SUM(rg.total_rows), 0) AS DECIMAL(5,2)) AS deleted_pct,
    SUM(rg.size_in_bytes) / 1024.0 / 1024.0 AS size_mb
FROM sys.column_store_row_groups rg
JOIN sys.indexes i ON rg.object_id = i.object_id AND rg.index_id = i.index_id
GROUP BY i.object_id, i.name, rg.state_desc
ORDER BY deleted_pct DESC;
-- Rebuild when deleted_pct > 10%
ALTER INDEX CCI_FactSales ON dbo.FactSales REBUILD;
```

---

## Key Metrics and Thresholds

| Metric | Source | Threshold / Action |
|---|---|---|
| Index fragmentation | `sys.dm_db_index_physical_stats` | Reorganize 5–30%; Rebuild > 30% |
| Unused index (reads = 0) | `sys.dm_db_index_usage_stats` | Review candidates; drop if write-heavy and never read |
| Missing index impact score | `sys.dm_db_missing_index_group_stats` | Implement if impact > 100,000 |
| Indexes per OLTP table | `sys.indexes` | Target < 10–15 non-clustered |
| Columnstore deleted rows | `sys.column_store_row_groups` | Rebuild if deleted > 10% |
| Composite index key size | `sys.indexes` | Keep < 900 bytes |

---

## References

- [Index types](references/index-types.md) — all index types with use cases and creation syntax
- [Maintenance scripts](references/maintenance-scripts.md) — complete fragmentation analysis and maintenance T-SQL
- [Design patterns](references/design-patterns.md) — index design patterns and anti-patterns
- [Examples](examples/examples.md) — fragmentation assessment, evaluating missing index recommendations, finding redundant indexes, designing indexes for a query
