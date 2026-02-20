# SQL Server Query Optimization Research

## Overview
Query optimization in SQL Server is the process of identifying and improving the performance of T-SQL queries. The goal is to minimize resource consumption (CPU, Memory, I/O) and reduce execution time.

## Execution Plans
- **Graphical Execution Plans:** Visual representation of the steps taken by the SQL Server Query Optimizer. Accessible in SSMS and Azure Data Studio.
- **Key Operators to Watch:**
    - **Index Scan vs. Index Seek:** Seek is generally preferred as it navigates the B-tree; Scan reads the entire index.
    - **Table Scan:** Indicates a lack of a useful clustered index or a query that needs too much data.
    - **Key Lookup / RID Lookup:** Occurs when a non-clustered index doesn't "cover" all columns requested, forcing a trip back to the data pages.
    - **Sort & Hash Match:** Can be expensive and may spill to `tempdb` if memory is insufficient.

## Statistics
- **Importance:** The Optimizer uses statistics (histograms, density vectors) to estimate the number of rows (cardinality) and choose the best plan.
- **Best Practices:**
    - Enable `AUTO_CREATE_STATISTICS` and `AUTO_UPDATE_STATISTICS`.
    - Use `UPDATE STATISTICS [Table] WITH FULLSCAN` for critical tables where sampled statistics are insufficient.

## Indexing Strategies
- **Covering Indexes:** Including all columns needed by a query in the index (via `INCLUDE` clause) to avoid lookups.
- **Filtered Indexes:** Indexes with a `WHERE` clause to target specific data subsets (e.g., `WHERE IsActive = 1`).
- **Columnstore Indexes:** Optimized for analytical workloads (OLAP) and large data scans.

## Query Store
- **Functionality:** Captures query history, execution plans, and runtime statistics.
- **Use Cases:** Identifying plan regressions, forcing better plans, and analyzing wait statistics at the query level.

## Common Performance Issues
- **Parameter Sniffing:** When a plan optimized for one set of parameters is used for another, causing poor performance. Solutions include `OPTION (RECOMPILE)` or `OPTIMIZE FOR`.
- **SARGability:** Writing queries so they can use indexes (Search ARGumentable). Avoid functions on columns in the `WHERE` clause (e.g., `WHERE YEAR(OrderDate) = 2023` is not SARGable).
- **Implicit Conversions:** Occur when data types don't match (e.g., comparing `VARCHAR` to `NVARCHAR`), potentially preventing index usage.

## Best Practices (2024-2025)
- **Intelligent Query Processing (IQP):** Leverage features like Memory Grant Feedback, Adaptive Joins, and Approximate Query Processing available in newer SQL Server versions.
- **Query Store Hints:** Apply hints to queries without changing code.
- **Parallelism:** Monitor `MAXDOP` (Maximum Degree of Parallelism) and "Cost Threshold for Parallelism" to prevent small queries from being over-parallelized.

## References
- [SQL Server Execution Plans (Grant Fritchey)](https://www.red-gate.com/library/sql-server-execution-plans-3rd-edition)
- [Microsoft Documentation: Query Tuning](https://learn.microsoft.com/en-us/sql/relational-databases/performance/query-tuning-and-execution-feedback-concepts)
- [Brent Ozar: Parameter Sniffing Guide](https://www.brentozar.com/sql-server-parameter-sniffing/)
