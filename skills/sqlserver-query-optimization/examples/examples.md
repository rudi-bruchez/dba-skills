# SQL Server Query Optimization — Examples

---

## Scenario 1: Slow Query Analysis — Report Taking 45 Minutes

**Context:** A monthly financial summary report that previously ran in 8 minutes now takes 45 minutes. No schema changes were deployed.

**Step 1 — Find the query in plan cache**
```sql
SELECT TOP 5
    qs.total_elapsed_time / qs.execution_count AS avg_elapsed_us,
    qs.total_logical_reads / qs.execution_count AS avg_reads,
    SUBSTRING(st.text, 1, 400) AS query_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
ORDER BY avg_elapsed_us DESC;
-- Result: usp_MonthlyFinancialSummary — avg 2,700,000,000 us (45 min), avg_reads 18,500,000
```

**Step 2 — Get the execution plan and check statistics**
```sql
SET STATISTICS IO ON;
EXEC dbo.usp_MonthlyFinancialSummary @month = '2024-01';
SET STATISTICS IO OFF;
-- Result: Table 'Transactions'. Scan count 1, logical reads 18,547,208
-- Estimated rows: 1,200  Actual rows: 4,800,000 (4000x off)
```

**Step 3 — Check statistics age**
```sql
SELECT s.name AS stat_name, sp.last_updated, sp.rows, sp.rows_sampled,
       sp.modification_counter
FROM sys.stats s
CROSS APPLY sys.dm_db_stats_properties(s.object_id, s.stats_id) sp
WHERE OBJECT_NAME(s.object_id) = 'Transactions'
ORDER BY sp.modification_counter DESC;
-- Result: last_updated = 2024-01-03, modification_counter = 22,400,000
-- Statistics are 3 months old; 22 million row changes since last update
```

**Resolution:**
```sql
-- Update statistics with full scan on the high-modification table
UPDATE STATISTICS dbo.Transactions WITH FULLSCAN;

-- Flush the cached plan so the next execution gets a fresh compile
DBCC FREEPROCCACHE;  -- Use with care on production (removes ALL cached plans)
-- Better: target the specific plan handle
DECLARE @plan_handle VARBINARY(64) = <plan_handle from query stats>;
DBCC FREEPROCCACHE(@plan_handle);
```

After statistics update, the report runs in 7 minutes — back to normal. Schedule automatic full-scan statistics updates weekly for high-churn tables.

---

## Scenario 2: Missing Index — Order Search Taking 12 Seconds

**Context:** An order search function in the customer portal is timing out. Users enter a customer ID and date range; the result should return in under 200 ms.

**Query:**
```sql
SELECT order_id, order_date, status, total_amount
FROM dbo.Orders
WHERE customer_id = @cust_id AND order_date BETWEEN @start_date AND @end_date;
```

**Step 1 — Check execution plan for missing index warning**
The plan shows:
- Clustered Index Scan on `dbo.Orders` (reading 4.2 million rows)
- Missing Index warning: `(customer_id, order_date) INCLUDE (status, total_amount)`
- Estimated improvement: 94.7%

**Step 2 — Check the missing index DMV**
```sql
SELECT ROUND(migs.avg_total_user_cost * migs.avg_user_impact
             * (migs.user_seeks + migs.user_scans), 0) AS impact_score,
       migs.user_seeks, migs.avg_user_impact,
       mid.equality_columns, mid.inequality_columns, mid.included_columns
FROM sys.dm_db_missing_index_details mid
JOIN sys.dm_db_missing_index_groups mig ON mid.index_handle = mig.index_handle
JOIN sys.dm_db_missing_index_group_stats migs ON mig.index_group_handle = migs.group_handle
WHERE OBJECT_NAME(mid.object_id) = 'Orders';
-- impact_score: 8,450,000 — very high
```

**Step 3 — Verify no existing index already covers this**
```sql
SELECT i.name, COL_NAME(ic.object_id, ic.column_id) AS col, ic.key_ordinal, ic.is_included_column
FROM sys.indexes i
JOIN sys.index_columns ic ON i.object_id = ic.object_id AND i.index_id = ic.index_id
WHERE OBJECT_NAME(i.object_id) = 'Orders'
ORDER BY i.name, ic.key_ordinal;
-- No existing index on customer_id + order_date
```

**Step 4 — Create the index**
```sql
CREATE NONCLUSTERED INDEX IX_Orders_Customer_OrderDate
ON dbo.Orders (customer_id ASC, order_date ASC)
INCLUDE (status, total_amount)
WITH (ONLINE = ON, FILLFACTOR = 85, SORT_IN_TEMPDB = ON);
```

**Result:** Query execution drops from 12 seconds to 8 ms. Logical reads drop from 480,000 to 12.

---

## Scenario 3: Parameter Sniffing — Stored Procedure Slow for Some Users

**Context:** `usp_GetCustomerOrders` runs in 50 ms for most customers but takes 8 minutes for a few large corporate accounts. The same stored procedure is called by all users.

**Step 1 — Check execution time variance**
```sql
SELECT qs.execution_count,
       qs.total_elapsed_time / qs.execution_count AS avg_elapsed_us,
       qs.max_elapsed_time AS max_elapsed_us,
       qs.min_elapsed_time AS min_elapsed_us,
       qs.max_elapsed_time / NULLIF(qs.min_elapsed_time, 0) AS variance_ratio,
       SUBSTRING(st.text, 1, 200) AS query_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
WHERE st.text LIKE '%usp_GetCustomerOrders%'
ORDER BY qs.max_elapsed_time DESC;
-- avg: 50,000 us  max: 480,000,000 us  ratio: 9,600x variance
```

**Step 2 — Look at the cached plan**
```sql
SELECT qp.query_plan
FROM sys.dm_exec_procedure_stats ps
CROSS APPLY sys.dm_exec_query_plan(ps.plan_handle) qp
WHERE OBJECT_NAME(ps.object_id) = 'usp_GetCustomerOrders';
-- Plan shows Nested Loops join — optimal for customer_id = 12345 (5 orders)
-- But customer_id = 1001 has 820,000 orders — Nested Loops is disastrous for this volume
```

**Root cause:** The plan was compiled when a small customer_id was used first. The nested loops plan works for customers with few orders but is catastrophic for large accounts.

**Fix — Add OPTION (RECOMPILE) to the query inside the procedure** (if the procedure runs infrequently enough):
```sql
ALTER PROCEDURE dbo.usp_GetCustomerOrders (@cust_id INT, @start_date DATE, @end_date DATE)
AS
BEGIN
    SELECT order_id, order_date, total_amount, status
    FROM dbo.Orders
    WHERE customer_id = @cust_id
      AND order_date BETWEEN @start_date AND @end_date
    OPTION (RECOMPILE);  -- Fresh plan per execution; optimal for high-variance parameters
END;
```

**Alternative — OPTIMIZE FOR UNKNOWN** (if the procedure runs frequently and recompile is too expensive):
```sql
    OPTION (OPTIMIZE FOR (@cust_id UNKNOWN));
    -- Uses average statistics; not optimal for any case, but decent for all cases
```

---

## Scenario 4: Implicit Conversion — Index Not Being Used

**Context:** A login query is scanning 2 million rows despite an index on the `username` column. The query runs in 3 seconds instead of the expected sub-millisecond.

**Query:**
```sql
-- From the application (Java/Hibernate):
SELECT user_id, email, password_hash FROM dbo.Users WHERE username = N'alice@example.com';
-- Note: N prefix = NVARCHAR parameter
-- Column definition: username VARCHAR(200)
```

**Step 1 — Find in plan cache**
```sql
SELECT SUBSTRING(st.text, 1, 300) AS query_text, qp.query_plan
FROM sys.dm_exec_cached_plans cp
CROSS APPLY sys.dm_exec_sql_text(cp.plan_handle) st
CROSS APPLY sys.dm_exec_query_plan(cp.plan_handle) qp
WHERE CAST(qp.query_plan AS NVARCHAR(MAX)) LIKE '%CONVERT_IMPLICIT%'
  AND st.text LIKE '%Users%';
```

The plan shows `CONVERT_IMPLICIT(nvarchar(200), [username], 0)` on the seek predicate — SQL converts every `varchar` column value to `nvarchar` to match the `nvarchar` parameter, causing a full index scan.

**Fix A — Change the Hibernate/JDBC type mapping** to send a `VARCHAR` parameter instead of `NVARCHAR`. This is the preferred fix.

**Fix B — Convert the column to NVARCHAR** (if the application cannot be changed):
```sql
ALTER TABLE dbo.Users ALTER COLUMN username NVARCHAR(200) NOT NULL;
-- Rebuild the index afterward
ALTER INDEX IX_Users_Username ON dbo.Users REBUILD;
```

**Result:** After changing the column type, logical reads drop from 2,000,000 to 3. Query time: 1 ms.

---

## Scenario 5: Query Regression — Stored Procedure Slow After Patch

**Context:** After applying SQL Server CU21, `usp_CalculateCommission` regressed from 200 ms to 45 seconds. Rollback is not feasible.

**Step 1 — Identify the regression using Query Store**
```sql
-- Query Store must be enabled on the database
SELECT qt.query_sql_text, q.query_id, p.plan_id,
       rs.avg_duration / 1000.0 AS avg_ms,
       rs.last_execution_time
FROM sys.query_store_query q
JOIN sys.query_store_query_text qt ON q.query_text_id = qt.query_text_id
JOIN sys.query_store_plan p ON q.query_id = p.query_id
JOIN sys.query_store_runtime_stats rs ON p.plan_id = rs.plan_id
WHERE qt.query_sql_text LIKE '%CalculateCommission%'
ORDER BY rs.avg_duration DESC;
-- Result: query_id = 187, plan_id = 12 — avg 45,000 ms (current, bad plan)
--         query_id = 187, plan_id = 5  — avg 200 ms (previous good plan)
```

**Step 2 — Compare the two plans in SSMS** by clicking the "Regressed Queries" report in Query Store (SSMS → Database → Query Store → Regressed Queries). The old plan (plan_id 5) used a Merge Join; the new plan (plan_id 12) uses a Hash Match with a TempDB spill.

**Step 3 — Force the known-good plan**
```sql
EXEC sys.sp_query_store_force_plan @query_id = 187, @plan_id = 5;
-- This takes effect immediately — no code changes, no deployment
```

**Step 4 — Verify**
```sql
SELECT rs.avg_duration / 1000.0 AS avg_ms, rs.count_executions
FROM sys.query_store_plan p
JOIN sys.query_store_runtime_stats rs ON p.plan_id = rs.plan_id
WHERE p.query_id = 187
ORDER BY rs.last_execution_time DESC;
-- avg_ms should return to ~200 ms after forcing
```

**Long-term:** Open a support case with Microsoft if the CU caused a regression. The forced plan is a temporary workaround; the root cause should be addressed (usually stale statistics or a cardinality estimator change in the new CU).
