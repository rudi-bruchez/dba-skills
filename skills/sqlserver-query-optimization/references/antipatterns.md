# SQL Server T-SQL Anti-Patterns

Common query patterns that cause performance problems, with before/after rewrites.

---

## 1. Function on Column in WHERE (Non-SARGable)

**Problem:** Applying a function to a column in a WHERE clause prevents the optimizer from using an index seek on that column.

**Before (index cannot be used for a seek):**
```sql
SELECT order_id, total_amount
FROM dbo.Orders
WHERE YEAR(order_date) = 2024;
-- Causes: Clustered Index Scan on entire table
```

**After (SARGable — index seek possible):**
```sql
SELECT order_id, total_amount
FROM dbo.Orders
WHERE order_date >= '2024-01-01' AND order_date < '2025-01-01';
-- Requires: index on order_date (e.g., as part of a covering index)
```

**Other non-SARGable patterns:**
```sql
-- Bad:  WHERE LEFT(last_name, 3) = 'Smi'
-- Good: WHERE last_name LIKE 'Smi%'

-- Bad:  WHERE CONVERT(VARCHAR, order_id) = '12345'
-- Good: WHERE order_id = 12345

-- Bad:  WHERE ISNULL(status, 'Unknown') = 'Active'
-- Good: WHERE status = 'Active' (add separate handling for NULLs if needed)

-- Bad:  WHERE LEN(description) > 100
-- Good: Add a persisted computed column: description_len AS LEN(description)
--       Then index on the computed column and filter on it
```

---

## 2. SELECT * (Column Over-Fetching)

**Problem:** Selecting all columns forces SQL Server to read more data than needed, increases network transfer, blocks columnstore usage, and causes key lookups when a covering index exists.

**Before:**
```sql
SELECT * FROM dbo.Customers WHERE country = 'US';
```

**After:**
```sql
SELECT customer_id, name, email, phone
FROM dbo.Customers
WHERE country = 'US';
-- Now a covering index on (country) INCLUDE (name, email, phone) can fully satisfy this
```

---

## 3. Implicit Type Conversion

**Problem:** When the data types of the column and the comparison value differ, SQL Server converts the column value for every row — preventing index seeks.

**Before (causes index scan — CONVERT_IMPLICIT in plan):**
```sql
-- Column: customer_id VARCHAR(20)
-- Parameter: @cust_id INT
SELECT * FROM dbo.Orders WHERE customer_id = @cust_id;
-- SQL converts customer_id to INT for every row
```

**After:**
```sql
-- Fix A: Match the parameter type to the column type
DECLARE @cust_id VARCHAR(20) = '10042';
SELECT * FROM dbo.Orders WHERE customer_id = @cust_id;

-- Fix B: Alter the column type (preferred long-term)
ALTER TABLE dbo.Orders ALTER COLUMN customer_id INT NOT NULL;
```

**Common implicit conversion scenarios:**
- `NVARCHAR` column compared to `VARCHAR` parameter
- `INT` column compared to `VARCHAR` literal
- `DATETIME` column compared to `DATE` parameter
- Application ORM sending wrong data types (fix at the application layer)

---

## 4. OR on Indexed Columns

**Problem:** An `OR` between conditions on different columns often prevents index usage — the optimizer may choose a full scan instead.

**Before (may cause a scan):**
```sql
SELECT order_id, customer_id, total_amount
FROM dbo.Orders
WHERE customer_id = 42 OR sales_rep_id = 99;
```

**After (each branch can use its own index via `UNION ALL`):**
```sql
SELECT order_id, customer_id, total_amount
FROM dbo.Orders WHERE customer_id = 42
UNION ALL
SELECT order_id, customer_id, total_amount
FROM dbo.Orders WHERE sales_rep_id = 99
  AND customer_id <> 42;  -- Exclude rows already returned by first branch
-- Requires indexes on both customer_id and sales_rep_id
```

---

## 5. Leading Wildcard LIKE

**Problem:** A `LIKE '%value'` or `LIKE '%value%'` pattern cannot use a B-tree index because the matching side is unknown.

**Before (causes full scan):**
```sql
SELECT * FROM dbo.Products WHERE description LIKE '%wireless%';
```

**After — Full-Text Search (if full-text index exists):**
```sql
SELECT * FROM dbo.Products WHERE CONTAINS(description, 'wireless');
-- Requires: CREATE FULLTEXT INDEX ON dbo.Products (description)
```

**After — leading wildcard only (accept the scan; limit with other predicates):**
```sql
SELECT * FROM dbo.Products
WHERE category_id = 5 AND description LIKE '%wireless%';
-- category_id can use an index to reduce the rows scanned
```

---

## 6. Scalar UDF in SELECT List

**Problem:** A scalar user-defined function (UDF) called for each row forces row-by-row "RBAR" execution and prevents parallelism.

**Before (executes row-by-row, disables parallelism):**
```sql
SELECT order_id, dbo.fn_GetCustomerTier(customer_id) AS tier
FROM dbo.Orders;
```

**After — inline table-valued function (iTVF):**
```sql
CREATE OR ALTER FUNCTION dbo.fn_GetCustomerTier_Inline(@customer_id INT)
RETURNS TABLE
AS RETURN (
    SELECT CASE
        WHEN total_spend > 100000 THEN 'Platinum'
        WHEN total_spend > 10000  THEN 'Gold'
        ELSE 'Standard'
    END AS tier
    FROM dbo.CustomerMetrics WHERE customer_id = @customer_id
);

SELECT o.order_id, t.tier
FROM dbo.Orders o
CROSS APPLY dbo.fn_GetCustomerTier_Inline(o.customer_id) t;
-- iTVFs are inlined into the query plan — parallelism and index usage work normally
```

---

## 7. Cursor over Large Result Sets

**Problem:** Cursors process rows one at a time — extremely slow for large datasets.

**Before (cursor — may take hours on large tables):**
```sql
DECLARE @order_id INT, @customer_id INT;
DECLARE order_cursor CURSOR FOR SELECT order_id, customer_id FROM dbo.Orders;
OPEN order_cursor;
FETCH NEXT FROM order_cursor INTO @order_id, @customer_id;
WHILE @@FETCH_STATUS = 0
BEGIN
    UPDATE dbo.Orders SET last_processed = GETDATE() WHERE order_id = @order_id;
    FETCH NEXT FROM order_cursor INTO @order_id, @customer_id;
END;
CLOSE order_cursor; DEALLOCATE order_cursor;
```

**After (set-based operation — seconds):**
```sql
UPDATE dbo.Orders SET last_processed = GETDATE();
-- Or with a filter:
UPDATE dbo.Orders SET last_processed = GETDATE() WHERE last_processed IS NULL;
```

---

## 8. Dynamic SQL Without sp_executesql

**Problem:** Building dynamic SQL with string concatenation generates a unique query string for each parameter value. Each unique string creates a new plan cache entry.

**Before (plan cache pollution — one plan per unique value):**
```sql
DECLARE @sql NVARCHAR(500);
SET @sql = 'SELECT * FROM dbo.Orders WHERE customer_id = ' + CAST(@cust_id AS VARCHAR);
EXEC(@sql);
```

**After (parameterized — one shared plan):**
```sql
DECLARE @sql NVARCHAR(500) = N'SELECT * FROM dbo.Orders WHERE customer_id = @cid';
EXEC sp_executesql @sql, N'@cid INT', @cid = @cust_id;
```

---

## 9. Missing JOIN Index (Causing Hash or Merge on Unindexed Column)

**Problem:** Joining two large tables on a column with no index forces a hash join with large memory grant.

**Before (hash join, large memory grant, possible TempDB spill):**
```sql
SELECT o.order_id, oi.product_id, oi.quantity
FROM dbo.Orders o
JOIN dbo.OrderItems oi ON o.order_id = oi.order_id
WHERE o.status = 'Pending';
-- If dbo.OrderItems has no index on order_id, SQL scans the entire table
```

**After (add index on the FK column):**
```sql
CREATE INDEX IX_OrderItems_OrderId ON dbo.OrderItems (order_id)
INCLUDE (product_id, quantity);

-- Now SQL can use Nested Loops (seek on OrderItems for each Order)
-- or Merge Join (both inputs sorted on order_id)
```

---

## 10. Aggregating Before Joining (Missed Optimization)

**Problem:** Aggregating inside a subquery or CTE after joining inflates the dataset being processed.

**Before:**
```sql
SELECT c.name, o.total_orders, o.total_spend
FROM dbo.Customers c
JOIN (
    SELECT customer_id,
           COUNT(*) AS total_orders,
           SUM(total_amount) AS total_spend
    FROM dbo.Orders
    GROUP BY customer_id
) o ON c.customer_id = o.customer_id;
```

This is actually a correct pattern — pre-aggregating in the derived table is often better than joining all rows then aggregating. Review the execution plan to confirm. The anti-pattern to avoid is joining first, then filtering:

**Before (join everything, then filter — wastes I/O):**
```sql
SELECT c.name, o.order_id
FROM dbo.Customers c
JOIN dbo.Orders o ON c.customer_id = o.customer_id
WHERE c.country = 'US' AND o.status = 'Pending';
-- If the filter on c.country eliminates most customers, push that filter early
```

**After (filter early):**
```sql
-- Let the optimizer handle this with SARGable predicates and indexes
-- If optimizer makes a wrong choice, use indexed views or CTEs to pre-filter
WITH PendingOrders AS (
    SELECT order_id, customer_id FROM dbo.Orders WHERE status = 'Pending'
)
SELECT c.name, po.order_id
FROM dbo.Customers c
JOIN PendingOrders po ON c.customer_id = po.customer_id
WHERE c.country = 'US';
```
