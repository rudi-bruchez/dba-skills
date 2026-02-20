# Execution Plan Guide

Graphical execution plans are the most important tool for understanding how SQL Server executes a query.

## Key Operators to Watch

| Operator | Symbol | Description | Warning Threshold |
|---|---|---|---|
| **Index Seek** | (B-tree Seek) | Navigates the index to find a specific range of values. This is generally the most efficient access method. | Seek should avoid high "Estimated Number of Rows" if unexpected. |
| **Index Scan** | (B-tree Scan) | Reads the entire index. Can be efficient for small tables or when many rows are needed. | Scan of a large table often indicates a missing index. |
| **Table Scan** | (Heap Scan) | Reads all pages in a heap table (no clustered index). | Table Scans are generally inefficient for large tables. Create a clustered index! |
| **Clustered Index Scan** | (B-tree Scan) | Reads all pages in the table. Similar to a table scan but on a clustered index. | Scan of a large table often indicates a missing index. |
| **Key Lookup / RID Lookup** | (B-tree Seek) | Navigates the clustered index to fetch additional columns not in the non-clustered index. | If lookups are frequent, create a "Covering Index." |
| **Nested Loop Join** | (Join) | For each row in the outer table, search for a match in the inner table. Efficient for small result sets. | Inefficient for large inner tables without a good index. |
| **Hash Match (Join/Agg)** | (Join/Agg) | Builds a hash table in memory to perform the join or aggregation. Efficient for large result sets. | Can "spill" to `tempdb` if memory is insufficient, causing major performance loss. |
| **Sort** | (Sort) | Sorts the input data. Can be expensive. | Sorting should be avoided if possible by using indexes that provide the requested order. |

## Execution Plan Tips

1.  **Look for High-Cost Operators:** SSMS highlights the "Cost %" for each operator. Focus on the most expensive ones.
2.  **Check for Warnings:** A yellow exclamation mark (!) indicates a potential issue, such as "Missing Statistics" or "TempDB Spilling."
3.  **Estimated vs. Actual Rows:** If there is a large discrepancy, statistics may be outdated or inaccurate.
4.  **Data Type Mismatches:** Look for "Implicit Conversion" warnings, which prevent SQL Server from using an index effectively.

## References
- [SQL Server Execution Plans (Grant Fritchey)](https://www.red-gate.com/library/sql-server-execution-plans-3rd-edition)
- [Microsoft Documentation: Query Tuning](https://learn.microsoft.com/en-us/sql/relational-databases/performance/query-tuning-and-execution-feedback-concepts)
