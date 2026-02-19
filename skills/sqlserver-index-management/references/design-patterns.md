# Index Design Patterns and Anti-Patterns

## Core Design Principles

1. **Index for the query, not the table.** Identify the most frequent and expensive queries; index to satisfy them.
2. **Equality columns before inequality.** In composite indexes: `WHERE country = 'US' AND age > 30` → key: `(country, age)`.
3. **Selectivity determines usefulness.** High-selectivity columns (many distinct values) benefit most from indexes.
4. **Every index has a write cost.** Each non-clustered index adds overhead to INSERT, UPDATE, DELETE, and MERGE operations.

---

## Pattern 1: Covering Index

Eliminates key lookups by including all columns a query needs within the index.

**Problem query:**
```sql
-- Causes a key lookup (bookmark lookup) for every row found
SELECT name, email, status
FROM dbo.Customers
WHERE country = 'US' AND account_type = 'Premium';
```

**Without covering index:** Optimizer does an index seek on `(country, account_type)`, then a key lookup per row to fetch `name`, `email`, `status`.

**Solution:**
```sql
CREATE NONCLUSTERED INDEX IX_Customers_Country_Type
ON dbo.Customers (country, account_type)
INCLUDE (name, email, status);  -- Not in the key; no key lookup needed
```

**Rule:** Include all `SELECT` columns that are not seek predicates as `INCLUDE` columns. Keep key columns to seek predicates only.

---

## Pattern 2: Filtered Index

Partial index covering only a subset of rows — smaller, faster, cheaper to maintain.

**When to use:**
- Queries always filter on a specific value (status, is_active, region).
- The filtered subset is a small fraction of the table (< 10–20%).
- Sparse columns: index only non-NULL values.

```sql
-- Only active orders (most order queries target active orders)
CREATE NONCLUSTERED INDEX IX_Orders_Active
ON dbo.Orders (customer_id, order_date)
INCLUDE (total_amount, ship_address)
WHERE status IN ('Pending', 'Processing');

-- Only non-NULL email (allows multiple NULLs in unique index)
CREATE UNIQUE NONCLUSTERED INDEX UX_Customers_Email
ON dbo.Customers (email)
WHERE email IS NOT NULL;
```

**Limitation:** Filtered indexes are not used by queries that don't include the filter predicate in their WHERE clause.

---

## Pattern 3: Composite Key Column Order

Column order in a composite index determines which query patterns it can support.

**Rule:** Place equality-predicate columns first, then range/inequality columns.

```sql
-- Query: WHERE department_id = 10 AND hire_date > '2020-01-01'
-- Correct order: equality first, range last
CREATE NONCLUSTERED INDEX IX_Employees_Dept_Hire
ON dbo.Employees (department_id, hire_date)
INCLUDE (employee_name, salary);

-- Wrong: range first prevents efficient seek on department_id
-- CREATE NONCLUSTERED INDEX IX_Employees_Hire_Dept
-- ON dbo.Employees (hire_date, department_id)  -- department_id cannot be efficiently sought
```

---

## Pattern 4: Columnstore for Analytical Queries on OLTP Tables

Non-clustered columnstore index allows a single table to serve both transactional (row-store) and analytical (column-store) queries.

```sql
-- Analytical query: total sales by product per month over two years
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_Sales_Analytics
ON dbo.Sales (sale_date, product_id, customer_id, quantity, unit_price, total_amount);
-- Batch mode processing: aggregation of millions of rows in seconds
-- Does not impact DML on the base table row-store
```

**When to use:**
- Reporting queries on OLTP tables that scan large ranges of data.
- Tables > 1 million rows with frequent aggregation queries.
- Workloads that can tolerate the index size (columnstore compresses well).

---

## Pattern 5: Clustered Index Key Selection

The clustered index key affects all non-clustered index sizes (it is the row locator).

| Key Type | Fragmentation | Non-clustered overhead | Recommendation |
|---|---|---|---|
| Auto-increment `INT` / `BIGINT` identity | None (sequential inserts) | Small (4–8 bytes) | Preferred for OLTP |
| `UNIQUEIDENTIFIER` (NEWID) | High (random) | Large (16 bytes) | Avoid; use `NEWSEQUENTIALID()` instead |
| `NEWSEQUENTIALID()` | Low (sequential) | Large (16 bytes) | Acceptable if GUID required |
| Composite natural key | Varies | Larger per non-clustered | Use only if queries range-scan on the natural key |

```sql
-- Sequential GUID to avoid fragmentation
CREATE TABLE dbo.Orders (
    order_id UNIQUEIDENTIFIER NOT NULL DEFAULT NEWSEQUENTIALID(),
    -- ...
    CONSTRAINT PK_Orders PRIMARY KEY CLUSTERED (order_id)
);
```

---

## Pattern 6: Index Consolidation (Combining Redundant Indexes)

Multiple overlapping indexes are a common anti-pattern after years of organic growth.

**Detection:**
```sql
-- Find indexes sharing the same leading key column
SELECT
    t.name                                          AS table_name,
    i1.name                                         AS index1,
    i2.name                                         AS index2,
    STUFF((SELECT ', ' + COL_NAME(ic.object_id, ic.column_id)
           FROM sys.index_columns ic
           WHERE ic.object_id = i1.object_id AND ic.index_id = i1.index_id
           ORDER BY ic.key_ordinal FOR XML PATH('')), 1, 2, '') AS index1_keys,
    STUFF((SELECT ', ' + COL_NAME(ic.object_id, ic.column_id)
           FROM sys.index_columns ic
           WHERE ic.object_id = i2.object_id AND ic.index_id = i2.index_id
           ORDER BY ic.key_ordinal FOR XML PATH('')), 1, 2, '') AS index2_keys
FROM sys.indexes i1
JOIN sys.indexes i2 ON i1.object_id = i2.object_id AND i1.index_id < i2.index_id
JOIN sys.tables t ON i1.object_id = t.object_id
WHERE i1.type > 0 AND i2.type > 0
  AND EXISTS (
      SELECT 1 FROM sys.index_columns ic1
      JOIN sys.index_columns ic2
          ON ic1.column_id = ic2.column_id
          AND ic2.object_id = i2.object_id AND ic2.index_id = i2.index_id AND ic2.key_ordinal = 1
      WHERE ic1.object_id = i1.object_id AND ic1.index_id = i1.index_id AND ic1.key_ordinal = 1
  );
```

**Resolution:** Drop the narrower index; widen the remaining one with `INCLUDE` columns to cover both query patterns.

---

## Anti-Patterns

### Anti-pattern 1: Over-indexing OLTP tables
- Every non-clustered index adds write overhead: INSERT must update all indexes; UPDATE must update affected indexes.
- Target: < 10–15 non-clustered indexes per OLTP table.
- Use `sys.dm_db_index_usage_stats` to identify indexes with zero reads and high `user_updates` — primary drop candidates.

### Anti-pattern 2: Indexing low-selectivity columns alone
- A non-clustered index on `gender` (2 distinct values) is useless — SQL will table scan instead.
- Low-selectivity columns work best as secondary columns in a composite index, or with a filtered index.

### Anti-pattern 3: Missing INCLUDE columns (key lookups)
- An index that causes a key lookup for every qualifying row can be worse than a table scan at scale.
- Identify key lookups in execution plans (shows as "Key Lookup (Clustered)" or "RID Lookup").
- Fix: add the fetched columns to the index as `INCLUDE` columns.

### Anti-pattern 4: GUID primary keys without NEWSEQUENTIALID()
- `NEWID()` generates random GUIDs → random page inserts → every insert potentially causes a page split → severe fragmentation.
- Replace with `NEWSEQUENTIALID()` (monotonically increasing within a server restart) or use `INT`/`BIGINT` IDENTITY.

### Anti-pattern 5: Rebuilding instead of reorganizing during business hours
- Online `REBUILD` holds schema modification locks at start and end of operation.
- During business hours: prefer `REORGANIZE` (truly online, interruptible) for 5–30% fragmented indexes.
- Save `REBUILD` for maintenance windows or use `RESUMABLE = ON`.

### Anti-pattern 6: Dropping unique indexes without checking constraint usage
- Unique non-clustered indexes often enforce business rules (unique email, unique order number).
- Always check `i.is_unique = 1` and review the constraint semantics before dropping.

---

## FILLFACTOR Guidelines

| Scenario | FILLFACTOR | Rationale |
|---|---|---|
| Sequential identity PK | 100 | Inserts always at end; no mid-page insertions |
| Balanced read/write OLTP | 80–85 | Leave room for inserts without page splits |
| Heavily updated / GUID PK | 70–80 | Random inserts; more free space to absorb splits |
| Read-only / static table | 100 | No inserts; no need for free space |
| Reporting / data warehouse | 100 | Bulk loads replace data; no incremental inserts |

```sql
-- Set instance-level default FILLFACTOR
EXEC sp_configure 'fill factor (%)', 80;
RECONFIGURE;

-- Override per-index (instance default ignored when specified explicitly)
ALTER INDEX IX_Orders_CustomerId ON dbo.Orders
REBUILD WITH (FILLFACTOR = 75, ONLINE = ON);
```
