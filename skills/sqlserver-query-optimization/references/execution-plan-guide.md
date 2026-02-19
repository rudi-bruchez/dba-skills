# SQL Server Execution Plan Guide

A reference for interpreting execution plan operators, warnings, and properties.

---

## How to Get an Execution Plan

```sql
-- Estimated plan (no query execution required)
SET SHOWPLAN_XML ON;
GO
SELECT * FROM dbo.Orders WHERE customer_id = 42 AND status = 'Pending';
GO
SET SHOWPLAN_XML OFF;

-- Actual plan (executes the query)
SET STATISTICS XML ON;
GO
SELECT * FROM dbo.Orders WHERE customer_id = 42 AND status = 'Pending';
GO
SET STATISTICS XML OFF;

-- I/O and time stats alongside the plan
SET STATISTICS IO ON;
SET STATISTICS TIME ON;
-- Run query here
SET STATISTICS IO OFF;
SET STATISTICS TIME OFF;
-- Look for: "Table 'Orders'. Scan count 1, logical reads 48521" -- 48521 reads is high
```

---

## Key Operators and What They Mean

### Table Scan
- Reads every row in the table.
- On a large table: always investigate. Indicates no useful index for the predicate.
- On a small table (< 1000 rows): often acceptable (optimizer may choose scan over seek).
- **Fix:** Add an index with the WHERE clause columns as key columns.

### Clustered Index Scan
- Reads every row via the clustered index (the table's physical storage).
- Functionally equivalent to Table Scan (heaps become Table Scans).
- Same investigation as Table Scan.

### Clustered Index Seek / Index Seek
- Navigates the B-tree to find matching rows efficiently.
- The target for all range and equality predicates.
- Low cost when the number of rows returned is small.

### Key Lookup (Clustered)
- A non-clustered index was used for the seek, but the non-clustered index does not contain all projected columns, so SQL goes back to the clustered index to retrieve them.
- **Cost:** One random I/O per matching row. Becomes expensive when many rows match.
- **Fix:** Add the projected columns as INCLUDE columns on the non-clustered index.
- Example:
  ```sql
  -- Query: SELECT name, email FROM dbo.Customers WHERE country = 'US'
  -- Index: CREATE INDEX IX_Country ON dbo.Customers (country) -- missing name, email
  -- Fix:   CREATE INDEX IX_Country ON dbo.Customers (country) INCLUDE (name, email)
  ```

### Hash Match (Join or Aggregation)
- Builds a hash table in memory from one input, then probes it with the other.
- Used when inputs are large and unordered.
- Memory grant required — look for "Spill" warnings on hash operators.
- **On join:** Usually indicates missing index on the join column of the larger table.
- **On aggregation:** May be appropriate for large GROUP BY operations.

### Nested Loops (Join)
- For each row in the outer input, seeks the inner input.
- Efficient when the outer input is small and there is an index on the inner join column.
- Inefficient when the outer input is large (many seeks against the inner table).

### Merge Join
- Requires both inputs sorted on the join key.
- Very efficient when inputs are pre-sorted (e.g., joining on clustered index keys).
- Optimizer may add a Sort operator to satisfy merge join — watch for this added Sort cost.

### Sort
- Explicit sorting of data — CPU and memory intensive.
- Memory grant required. Spills to TempDB if grant is insufficient.
- **Fix:** Index on the ORDER BY / GROUP BY columns if the sort appears on a large dataset.

### Parallelism (Gather Streams / Repartition Streams / Distribute Streams)
- Data flowing between parallel threads.
- Not always a problem — parallelism helps large operations.
- Investigate if accompanied by large CXPACKET/CXCONSUMER waits.

### Spool (Eager / Lazy / Index Spool)
- Temporary buffering of data in TempDB during query execution.
- Eager Spool: reads all data before producing output — often a Halloween protection mechanism.
- Index Spool: creates a temporary index — expensive; usually indicates a missing permanent index.

---

## Plan Warnings (Yellow Exclamation Marks in SSMS)

### Missing Index Warning
- The optimizer detected that an index would significantly improve this query.
- Follow the missing index workflow in SKILL.md.
- The warning includes the table and columns — use as a starting point, not a definitive prescription.

### Implicit Conversion Warning
- A data type mismatch forces the optimizer to convert values at runtime.
- Usually causes an index scan where a seek would be possible.
- Fix: match data types between column definition and the parameter/literal used in the predicate.
- Example: `WHERE varchar_col = @nvarchar_param` — SQL converts every row's value.

### No Statistics / Statistics Not Up To Date
- Cardinality estimates may be incorrect.
- Run: `UPDATE STATISTICS dbo.TableName WITH FULLSCAN;`

### Memory Grant Warning / Spill to TempDB
- The actual number of rows processed was much larger than estimated.
- Often caused by stale statistics or parameter sniffing.
- Update statistics. Check for parameter sniffing (see SKILL.md).

### Residual Predicate (Predicate vs. Seek Predicate in Seek operators)
- A Seek Predicate: applied efficiently on the index key (fast).
- A Predicate: applied after the seek, row by row (slower).
- Examine what columns are in each — extra predicate columns may need to be part of the index key.

---

## Reading Row Count Estimates vs. Actuals

A large gap between Estimated Rows and Actual Rows is the primary signal that plan quality is poor.

| Estimated Rows | Actual Rows | Interpretation |
|---|---|---|
| 1 | 500,000 | Statistics severely underestimate — stale or non-existent stats; or parameter sniffing |
| 500,000 | 1 | Statistics overestimate — optimizer chose a plan optimized for large results but few were returned |
| Close | Close | Optimizer had accurate information — plan is likely good |

**How to find the worst estimates:**
In SSMS, hover over each operator node to see EstimateRows and ActualRows. The `%` in the operator node shows relative cost within the plan.

---

## Reading SET STATISTICS IO Output

```
Table 'Orders'. Scan count 1, logical reads 48521, physical reads 0,
page server reads 0, read-ahead reads 0, ...
```

| Field | Meaning |
|---|---|
| Scan count | How many times the table was accessed (0 for index seeks, > 1 may indicate loop joins) |
| logical reads | Pages read from buffer pool — main cost indicator for I/O |
| physical reads | Pages read from disk (not in cache) |
| read-ahead reads | Pages pre-fetched by read-ahead mechanism |

**Target for OLTP queries:** logical reads < 1000.
**High logical reads with scan count > 1:** Nested loop join doing many inner table seeks — add index on the inner join column.
