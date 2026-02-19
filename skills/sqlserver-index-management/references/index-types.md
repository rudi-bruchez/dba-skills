# SQL Server Index Types — Reference

## Clustered Index

- Defines the physical storage order of table rows in a B-tree.
- One per table maximum (a table without one is a "heap").
- Row data is stored at the B-tree leaf level.

**Best on:** Primary key columns, range query columns, frequently joined foreign key columns.
**Avoid on:** Frequently updated columns (causes row movement), GUIDs with random values (causes fragmentation — use FILLFACTOR 70-80 if unavoidable).

```sql
CREATE CLUSTERED INDEX CIX_Orders_OrderId
ON dbo.Orders (order_id ASC)
WITH (FILLFACTOR = 95, SORT_IN_TEMPDB = ON, ONLINE = ON);
```

---

## Non-Clustered Index

- Separate B-tree structure with a row locator pointer (clustered key or RID) at the leaf level.
- Up to 999 per table; practical OLTP limit: < 10–15 to limit write overhead.
- Add `INCLUDE` columns to eliminate key lookups.

**Best on:** Columns in WHERE, JOIN ON, ORDER BY, and GROUP BY clauses.

```sql
CREATE NONCLUSTERED INDEX IX_Orders_CustomerId_OrderDate
ON dbo.Orders (customer_id ASC, order_date DESC)
INCLUDE (total_amount, status)        -- Covering: avoids key lookup
WHERE status IN ('Pending', 'Processing')  -- Filtered: partial index
WITH (
    FILLFACTOR = 80,
    PAD_INDEX = ON,
    SORT_IN_TEMPDB = ON,
    ONLINE = ON,                      -- Enterprise Edition only
    DATA_COMPRESSION = PAGE
)
ON [PRIMARY];
```

---

## Columnstore Index (SQL 2012+)

Column-oriented storage with ~10:1 compression. Uses batch mode processing for high-throughput aggregations.

### Clustered Columnstore Index (CCI)
- Entire table stored as columnstore; no separate row store.
- Best for data warehouse fact tables and reporting tables.

```sql
CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactSales
ON dbo.FactSales
WITH (DATA_COMPRESSION = COLUMNSTORE, ONLINE = ON);
-- DATA_COMPRESSION options:
--   COLUMNSTORE: default (good balance)
--   COLUMNSTORE_ARCHIVE: higher ratio, slower access (archive/cold data)
```

### Non-Clustered Columnstore Index (NCCI)
- OLTP table retains row store for DML; columnstore handles analytics ("dual" access pattern).

```sql
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_Orders_Analytics
ON dbo.Orders (order_date, customer_id, product_id, quantity, unit_price, total_amount)
WHERE status = 'Completed';  -- Filtered NCCI (SQL 2016+)
```

**Rowgroup states to know:**
| State | Meaning |
|---|---|
| OPEN | Delta store (row mode), accumulating rows |
| CLOSED | Ready for compression; Tuple Mover will compress |
| COMPRESSED | Compressed columnstore rowgroup (~1,048,576 rows target) |
| TOMBSTONE | Being deleted (post-rebuild) |
| INVISIBLE | Being replaced during rebuild |

---

## Filtered Index

Non-clustered index with a `WHERE` clause. Smaller and faster than a full index for queries always scoped to specific values. Statistics are maintained only for the filtered rows.

```sql
-- Index only active accounts; much smaller than full table index
CREATE NONCLUSTERED INDEX IX_Accounts_Active
ON dbo.Accounts (account_type, last_activity_date)
INCLUDE (balance)
WHERE is_active = 1;
```

**When to use:**
- Queries always filter on a low-cardinality column (status, is_active, region).
- Sparse columns where NULL values are common and rarely queried.

---

## Unique Index

Enforces uniqueness. Can be clustered or non-clustered. Automatically created by `PRIMARY KEY` and `UNIQUE` constraints.

```sql
CREATE UNIQUE NONCLUSTERED INDEX UX_Customers_Email
ON dbo.Customers (email)
WHERE email IS NOT NULL;  -- Allows multiple NULLs (SQL filtered unique index)
```

---

## Full-Text Index

Enables linguistic search via `CONTAINS` and `FREETEXT`. Managed by the Full-Text Engine, separate from B-tree indexes. Requires a Full-Text Catalog.

```sql
-- Create catalog
CREATE FULLTEXT CATALOG FTC_Products AS DEFAULT;

-- Create full-text index on product description
CREATE FULLTEXT INDEX ON dbo.Products (description LANGUAGE 1033)
KEY INDEX PK_Products
ON FTC_Products
WITH CHANGE_TRACKING AUTO;

-- Query
SELECT product_id, name FROM dbo.Products
WHERE CONTAINS(description, '"stainless steel" AND bolt');
```

---

## XML Index

For `xml` data type columns. Primary XML index shreds XML for efficient querying. Secondary indexes optimize specific access patterns.

```sql
-- Primary XML index (must exist before secondary)
CREATE PRIMARY XML INDEX PX_Docs_XmlData
ON dbo.Documents (xml_data);

-- Secondary: PATH — optimize FOR XML PATH queries and .exist() with path expressions
CREATE XML INDEX SX_Docs_XmlData_PATH
ON dbo.Documents (xml_data)
USING XML INDEX PX_Docs_XmlData FOR PATH;

-- Secondary: VALUE — optimize value() and exist() with no path
CREATE XML INDEX SX_Docs_XmlData_VALUE
ON dbo.Documents (xml_data)
USING XML INDEX PX_Docs_XmlData FOR VALUE;

-- Secondary: PROPERTY — optimize multiple properties per row
CREATE XML INDEX SX_Docs_XmlData_PROPERTY
ON dbo.Documents (xml_data)
USING XML INDEX PX_Docs_XmlData FOR PROPERTY;
```

---

## Spatial Index

For `geography` and `geometry` data types.

```sql
CREATE SPATIAL INDEX SIX_Locations_GeoPoint
ON dbo.Locations (geo_point)
USING GEOGRAPHY_GRID
WITH (
    GRIDS = (LEVEL_1 = MEDIUM, LEVEL_2 = MEDIUM, LEVEL_3 = MEDIUM, LEVEL_4 = MEDIUM),
    CELLS_PER_OBJECT = 16
);
```

---

## Hash Index (In-Memory OLTP / Hekaton)

Only for memory-optimized tables. Equality lookups only — no range scans, no sorting.

```sql
-- Defined in CREATE TABLE for memory-optimized tables
CREATE TABLE dbo.SessionCache (
    session_id UNIQUEIDENTIFIER NOT NULL,
    user_data NVARCHAR(4000),
    CONSTRAINT PK_Session PRIMARY KEY NONCLUSTERED HASH (session_id)
        WITH (BUCKET_COUNT = 131072)  -- Must be power of 2; set to ~2x expected row count
) WITH (MEMORY_OPTIMIZED = ON, DURABILITY = SCHEMA_ONLY);
```

---

## Index Type Selection Summary

| Use Case | Index Type |
|---|---|
| Primary key, range queries | Clustered |
| WHERE / JOIN filter columns | Non-clustered |
| Avoid key lookups on high-read queries | Non-clustered with INCLUDE |
| Analytics on OLTP table | Non-clustered Columnstore |
| Data warehouse fact table | Clustered Columnstore |
| Sparse, targeted WHERE clause | Filtered |
| Linguistic search | Full-Text |
| XML data type queries | XML (Primary + Secondary) |
| Geography / geometry queries | Spatial |
| Memory-optimized, equality only | Hash |
