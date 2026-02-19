# SQL Server Index Management — Examples

## Example 1: Fragmentation Assessment and Maintenance Decision

**Scenario:** Weekly maintenance window; assess fragmentation across a 200 GB OLTP database.

**Step 1 — Run fragmentation check:**
```sql
SELECT
    OBJECT_NAME(i.object_id)                AS table_name,
    i.name                                  AS index_name,
    i.type_desc,
    ips.avg_fragmentation_in_percent,
    ips.page_count,
    CASE
        WHEN ips.avg_fragmentation_in_percent > 30 THEN 'REBUILD'
        WHEN ips.avg_fragmentation_in_percent > 5  THEN 'REORGANIZE'
        ELSE 'NONE'
    END AS recommended_action
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED') ips
JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
WHERE ips.page_count > 100 AND ips.index_id > 0
ORDER BY ips.avg_fragmentation_in_percent DESC;
```

**Sample output:**

| table_name | index_name | avg_fragmentation_pct | page_count | recommended_action |
|---|---|---|---|---|
| Orders | IX_Orders_CustomerId | 67.3 | 45200 | REBUILD |
| OrderItems | IX_Items_OrderId | 28.1 | 12800 | REORGANIZE |
| Products | IX_Products_Category | 3.2 | 880 | NONE |
| Customers | PK_Customers | 1.1 | 2200 | NONE |

**Decisions:**
- `IX_Orders_CustomerId` (67.3%): REBUILD. During maintenance window, use `ONLINE = ON` if Enterprise Edition.
- `IX_Items_OrderId` (28.1%): REORGANIZE. Can be done online during business hours if needed.
- Others: No action needed.

**Step 2 — Execute:**
```sql
-- Rebuild IX_Orders_CustomerId (67.3% fragmented)
ALTER INDEX IX_Orders_CustomerId ON dbo.Orders
REBUILD WITH (ONLINE = ON, FILLFACTOR = 80, SORT_IN_TEMPDB = ON);

-- Reorganize IX_Items_OrderId (28.1%)
ALTER INDEX IX_Items_OrderId ON dbo.OrderItems REORGANIZE;
-- Update statistics after reorganize (rebuild does this automatically)
UPDATE STATISTICS dbo.OrderItems WITH SAMPLE 30 PERCENT;
```

---

## Example 2: Evaluating a Missing Index Recommendation

**Scenario:** Missing index DMV reports a high-impact suggestion. Evaluate before creating.

**Query:**
```sql
SELECT TOP 5
    OBJECT_NAME(mid.object_id, mid.database_id)       AS table_name,
    migs.user_seeks,
    ROUND(migs.avg_total_user_cost * migs.avg_user_impact
          * (migs.user_seeks + migs.user_scans), 0)   AS impact_score,
    migs.avg_user_impact                              AS avg_pct_improvement,
    mid.equality_columns,
    mid.inequality_columns,
    mid.included_columns
FROM sys.dm_db_missing_index_details mid
JOIN sys.dm_db_missing_index_groups mig ON mid.index_handle = mig.index_handle
JOIN sys.dm_db_missing_index_group_stats migs ON mig.index_group_handle = migs.group_handle
WHERE mid.database_id = DB_ID()
ORDER BY impact_score DESC;
```

**Sample output:**

| table_name | user_seeks | impact_score | avg_pct_improvement | equality_columns | inequality_columns | included_columns |
|---|---|---|---|---|---|---|
| Orders | 48200 | 4,820,000 | 98.5 | [customer_id], [status] | [order_date] | [total_amount], [ship_city] |
| Customers | 12100 | 380,000 | 72.1 | [country] | NULL | [name], [email] |

**Evaluation of the top recommendation (Orders, impact 4,820,000):**

1. **Impact score 4,820,000 >> 100,000 threshold** → proceed.
2. **Check for overlapping existing indexes:**
```sql
SELECT i.name, COL_NAME(ic.object_id, ic.column_id) AS key_col, ic.key_ordinal, ic.is_included_column
FROM sys.indexes i
JOIN sys.index_columns ic ON i.object_id = ic.object_id AND i.index_id = ic.index_id
WHERE i.object_id = OBJECT_ID('dbo.Orders') AND i.type > 0
ORDER BY i.name, ic.key_ordinal;
-- Result: existing IX_Orders_CustomerId covers (customer_id) only — no status, no date
```
3. **No existing index covers** `(customer_id, status, order_date)` — creating is justified.
4. **Column order check:** equality columns first (`customer_id`, `status`), then inequality (`order_date`) — correct.
5. **Table currently has 8 non-clustered indexes** → under the 10–15 limit, safe to add.

**Create the index:**
```sql
CREATE NONCLUSTERED INDEX IX_Orders_Customer_Status_Date
ON dbo.Orders (customer_id, status, order_date)
INCLUDE (total_amount, ship_city)
WITH (ONLINE = ON, FILLFACTOR = 80, SORT_IN_TEMPDB = ON);
```

---

## Example 3: Finding and Dropping Unused Indexes

**Scenario:** A table has 18 non-clustered indexes. Identify candidates for removal.

**Query:**
```sql
SELECT
    i.name                                  AS index_name,
    ISNULL(ius.user_seeks, 0)               AS user_seeks,
    ISNULL(ius.user_scans, 0)               AS user_scans,
    ISNULL(ius.user_lookups, 0)             AS user_lookups,
    ISNULL(ius.user_updates, 0)             AS user_updates,
    ius.last_user_seek,
    'DROP INDEX ' + QUOTENAME(i.name) + ' ON dbo.Orders' AS drop_statement
FROM sys.indexes i
LEFT JOIN sys.dm_db_index_usage_stats ius
    ON i.object_id = ius.object_id AND i.index_id = ius.index_id AND ius.database_id = DB_ID()
WHERE i.object_id = OBJECT_ID('dbo.Orders')
  AND i.type > 0
  AND i.is_primary_key = 0
  AND i.is_unique = 0
ORDER BY (ISNULL(ius.user_seeks, 0) + ISNULL(ius.user_scans, 0) + ISNULL(ius.user_lookups, 0)) ASC,
         ISNULL(ius.user_updates, 0) DESC;
```

**Sample output (top drop candidates):**

| index_name | user_seeks | user_scans | user_lookups | user_updates | last_user_seek |
|---|---|---|---|---|---|
| IX_Orders_LegacyStatus | 0 | 0 | 0 | 482100 | NULL |
| IX_Orders_OldRegion | 0 | 12 | 0 | 481900 | NULL |
| IX_Orders_TempFilter | 0 | 0 | 0 | 481800 | 2021-03-15 |

**Evaluation:**
- `IX_Orders_LegacyStatus`: 0 reads, 482,100 updates — never read, high write cost. **Drop candidate.**
- `IX_Orders_OldRegion`: 12 scans only (likely from maintenance jobs), high updates. **Drop candidate after verifying the 12 scans are not business-critical.**
- `IX_Orders_TempFilter`: Zero reads since server restart; last seek was 2021 — outdated. **Drop candidate.**

**Important checks before dropping:**
- Confirm SQL Server has been running > 1 week to capture all workload patterns.
- Check if the index is referenced by any hints (`WITH (INDEX(...))`) in stored procedures or application code.
- Script the index to a file before dropping for rollback capability.

```sql
-- Drop after verification
DROP INDEX IX_Orders_LegacyStatus ON dbo.Orders;
DROP INDEX IX_Orders_TempFilter ON dbo.Orders;
```

---

## Example 4: Designing an Index for a Specific Query

**Scenario:** A new query is performing poorly. Design the optimal index.

**Query:**
```sql
-- Slow: ~8 seconds, full table scan
SELECT
    c.customer_id,
    c.full_name,
    c.email,
    SUM(o.total_amount) AS lifetime_value
FROM dbo.Customers c
JOIN dbo.Orders o ON c.customer_id = o.customer_id
WHERE c.country = 'Germany'
  AND c.account_status = 'Active'
  AND o.order_date >= '2024-01-01'
GROUP BY c.customer_id, c.full_name, c.email;
```

**Analysis:**
1. `dbo.Customers` filter: `country = 'Germany'` (equality) + `account_status = 'Active'` (equality). Columns to project: `customer_id`, `full_name`, `email`.
2. `dbo.Orders` filter: `customer_id` (JOIN, equality) + `order_date >= '2024-01-01'` (range). Column to aggregate: `total_amount`.

**Index for Customers:**
```sql
-- country, account_status → equality predicates in key
-- full_name, email → projected columns (include)
CREATE NONCLUSTERED INDEX IX_Customers_Country_Status
ON dbo.Customers (country, account_status)
INCLUDE (customer_id, full_name, email)
WHERE account_status = 'Active';  -- Filtered: if 'Active' is queried exclusively
```

**Index for Orders:**
```sql
-- customer_id → JOIN equality first; order_date → range last
-- total_amount → aggregated column (include)
CREATE NONCLUSTERED INDEX IX_Orders_Customer_Date
ON dbo.Orders (customer_id, order_date)
INCLUDE (total_amount);
```

**Result:** Query drops from 8 seconds to < 100 ms. Execution plan shows index seeks with no key lookups.

---

## Example 5: Columnstore Health Check and Rebuild Decision

**Scenario:** Columnstore index on a fact table has accumulated deleted rows from daily ETL updates.

**Health check:**
```sql
SELECT
    OBJECT_NAME(i.object_id)                        AS table_name,
    i.name                                          AS index_name,
    rg.state_desc,
    COUNT(*)                                        AS rowgroup_count,
    SUM(rg.total_rows)                              AS total_rows,
    SUM(rg.deleted_rows)                            AS deleted_rows,
    CAST(SUM(rg.deleted_rows) * 100.0
         / NULLIF(SUM(rg.total_rows), 0) AS DECIMAL(5,2)) AS deleted_pct
FROM sys.column_store_row_groups rg
JOIN sys.indexes i ON rg.object_id = i.object_id AND rg.index_id = i.index_id
WHERE i.object_id = OBJECT_ID('dbo.FactSales')
GROUP BY i.object_id, i.name, rg.state_desc;
```

**Output:**

| table_name | index_name | state_desc | rowgroup_count | total_rows | deleted_rows | deleted_pct |
|---|---|---|---|---|---|---|
| FactSales | CCI_FactSales | COMPRESSED | 240 | 251,658,240 | 38,600,000 | 15.3% |
| FactSales | CCI_FactSales | OPEN | 1 | 420,000 | 0 | 0.00% |
| FactSales | CCI_FactSales | CLOSED | 3 | 3,145,728 | 0 | 0.00% |

**Interpretation:** `deleted_pct = 15.3%` exceeds the 10% threshold. The OPEN and CLOSED rowgroups are normal (ETL loaded new rows, Tuple Mover not yet processed CLOSED).

**Action:**
```sql
-- Reorganize: compresses CLOSED rowgroups, removes deleted rows without full rebuild
ALTER INDEX CCI_FactSales ON dbo.FactSales REORGANIZE WITH (COMPRESS_ALL_ROW_GROUPS = ON);

-- If deleted_pct is > 30% or query performance requires full optimization:
ALTER INDEX CCI_FactSales ON dbo.FactSales REBUILD WITH (ONLINE = ON);
```

**Post-maintenance check:** Re-run the health query. Expect `deleted_pct` to drop to 0% and CLOSED rowgroups to become COMPRESSED.
