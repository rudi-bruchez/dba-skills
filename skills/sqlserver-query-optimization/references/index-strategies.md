# SQL Server Index Strategies for Query Optimization

---

## Covering Index Pattern

A covering index contains all columns needed by a query — both seek predicates (in the key) and projected columns (in INCLUDE). The result is that the optimizer can satisfy the entire query from the index without returning to the clustered index.

```sql
-- Query to optimize:
SELECT order_date, total_amount, status
FROM dbo.Orders
WHERE customer_id = 42 AND order_date >= '2024-01-01';

-- Non-covering index (causes Key Lookup):
CREATE INDEX IX_Orders_Customer ON dbo.Orders (customer_id, order_date);

-- Covering index (no Key Lookup):
CREATE INDEX IX_Orders_Customer_Covering ON dbo.Orders (customer_id, order_date)
INCLUDE (total_amount, status);
-- Rule: equality predicates first, inequality/range predicates last in key; projected columns in INCLUDE
```

**When to add INCLUDE columns vs. key columns:**
- Key columns: used in WHERE, JOIN ON, and ORDER BY clauses.
- INCLUDE columns: used only in the SELECT list (not for ordering or filtering).
- INCLUDE columns do not affect the sort order of the index; key columns do.

---

## Composite Index Column Order

The left-most key column must match the leading predicate for the index to be used for a seek (not a scan).

```sql
-- Index: (a, b, c)
-- Can seek on:  a=1                (leading column match)
-- Can seek on:  a=1 AND b=2        (leading two columns)
-- Can seek on:  a=1 AND b > 5      (range on b after equality on a)
-- Cannot seek:  b=2                (skips leading column a)
-- Cannot seek:  b=2 AND c=3        (skips a)
```

**Equality columns before inequality:**
```sql
-- Query: WHERE country = 'US' AND status = 'Active' AND order_date > '2024-01-01'
-- Optimal index:
CREATE INDEX IX_Optimal ON dbo.Orders (country, status, order_date);
-- country and status are equality, order_date is range — equality columns come first
```

---

## Filtered Index

A partial index covering only a subset of rows matching a WHERE condition. Smaller, maintained faster, used only when the query filter matches the index filter.

```sql
-- Index only pending orders (a small fraction of the total table)
CREATE INDEX IX_Orders_Pending
ON dbo.Orders (customer_id, order_date)
INCLUDE (total_amount)
WHERE status = 'Pending';

-- This query can use the filtered index:
SELECT customer_id, order_date, total_amount
FROM dbo.Orders
WHERE status = 'Pending' AND customer_id = 42;

-- This query CANNOT use the filtered index (status filter not included in WHERE):
SELECT customer_id, order_date FROM dbo.Orders WHERE customer_id = 42;
```

**Good candidates for filtered indexes:**
- Soft-delete pattern: `WHERE is_deleted = 0`
- Status column with one dominant value to exclude: `WHERE status <> 'Closed'`
- Nullable columns that are commonly queried for non-NULL: `WHERE email IS NOT NULL`

---

## Columnstore Indexes for Mixed Workloads

Add a non-clustered columnstore index to an OLTP table to accelerate analytical queries without affecting the row-store clustered index.

```sql
-- OLTP table with both transactional and reporting queries
-- Transactional queries use the existing B-tree indexes
-- Analytical queries (large aggregations) use the columnstore

CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_Orders_Analytics
ON dbo.Orders (sale_date, product_id, customer_id, quantity, unit_price, total_amount);

-- SQL Server uses batch mode processing with columnstore for GROUP BY aggregations
-- Typical speedup: 10x–100x for analytic scans vs. row-mode execution
```

---

## Unique Index as Constraint

Use a unique index when the business rule requires uniqueness. The index both enforces the constraint and enables efficient equality lookups.

```sql
-- Enforce unique email addresses (also speeds up login lookups)
CREATE UNIQUE NONCLUSTERED INDEX UIX_Customers_Email
ON dbo.Customers (email)
WHERE email IS NOT NULL;  -- Filtered unique allows multiple NULLs
```

---

## Index on Computed Columns

Index a computed column to make non-SARGable predicates SARGable.

```sql
-- Non-SARGable query: WHERE YEAR(order_date) = 2024
-- Cannot use any index on order_date directly

-- Solution: persisted computed column + index
ALTER TABLE dbo.Orders ADD order_year AS YEAR(order_date) PERSISTED;
CREATE INDEX IX_Orders_Year ON dbo.Orders (order_year);

-- Now this query can use the index:
SELECT * FROM dbo.Orders WHERE order_year = 2024;
-- Better alternative: WHERE order_date >= '2024-01-01' AND order_date < '2025-01-01'
```

---

## FILLFACTOR and Page Splits

FILLFACTOR controls how full each index page is when built (or rebuilt). Free space reserves room for inserts without immediate page splits.

```sql
-- Rebuild index with 80% fill factor (20% free for growth)
ALTER INDEX IX_Orders_Customer ON dbo.Orders
REBUILD WITH (FILLFACTOR = 80, ONLINE = ON);
```

**Choosing FILLFACTOR:**
- Sequential inserts (identity PK): FILLFACTOR = 100 (no splits; always appending)
- Random inserts (GUID PK): FILLFACTOR = 70–80 (many mid-page inserts)
- High-update volatile table: FILLFACTOR = 70–75
- Read-mostly: FILLFACTOR = 90–100

---

## Index on Foreign Keys

Foreign key columns that appear in JOIN conditions should almost always be indexed. SQL Server does not automatically create indexes for FK columns.

```sql
-- Table: dbo.OrderItems (order_id FK → dbo.Orders)
-- Deleting from dbo.Orders will scan dbo.OrderItems without an index:
CREATE INDEX IX_OrderItems_OrderId ON dbo.OrderItems (order_id);
```
