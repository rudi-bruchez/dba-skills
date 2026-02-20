# SQL Server Index Management Research

## Overview
Indexes are critical for performance, but they also incur a maintenance cost. Effective index management involves choosing the right types of indexes and maintaining them to avoid fragmentation and ensure statistics are up to date.

## Index Types
- **Clustered Index:** Defines the physical order of data in the table. Each table can have only one.
- **Non-Clustered Index:** A separate structure that points to the data rows. A table can have many.
- **Unique Index:** Ensures all values in the index are distinct.
- **Columnstore Index:** Optimized for large-scale data scans and analytical workloads.
- **Filtered Index:** An index that only includes a subset of rows based on a `WHERE` clause.
- **XML / Spatial / Full-Text Indexes:** Specialized indexes for specific data types.

## Index Fragmentation
- **Internal Fragmentation:** Empty space on index pages (can be caused by low Fill Factor or page splits).
- **External Fragmentation:** Index pages are out of physical order (caused by inserts/updates).
- **Identifying Fragmentation:** Use `sys.dm_db_index_physical_stats`.
    - **Reorganize:** Recommended for 5% to 30% fragmentation. Online operation, low resource usage.
    - **Rebuild:** Recommended for >30% fragmentation. Can be offline (standard edition) or online (enterprise edition). More resource-intensive.

## Fill Factor
- **Definition:** The percentage of space to fill on each leaf-level page during index creation or rebuild.
- **Best Practices:** Use 80-90% for tables with frequent inserts/updates to avoid page splits. Use 100 (or 0) for read-only tables.

## Missing & Unused Indexes
- **Missing Indexes:** Identified by the Query Optimizer. Can be queried via `sys.dm_db_missing_index_details`. Use with caution as too many indexes can slow down writes.
- **Unused Indexes:** Identified via `sys.dm_db_index_usage_stats`. Dropping unused indexes can improve write performance and reduce storage and maintenance overhead.

## Best Practices (2024-2025)
- **Automatic Index Management:** Use features like Azure SQL Database's automatic indexing if applicable.
- **Resumable Index Operations:** SQL Server 2017+ supports pausing and resuming index rebuilds.
- **Ordered Columnstore Indexes:** SQL Server 2022+ feature that can significantly improve performance for range-based queries.
- **Statistics Correlation:** Be aware that index maintenance affects statistics. Rebuilding an index updates statistics with `FULLSCAN`.

## Key T-SQL Commands
- `ALTER INDEX [IndexName] ON [TableName] REORGANIZE;`
- `ALTER INDEX [IndexName] ON [TableName] REBUILD;`
- `DROP INDEX [IndexName] ON [TableName];`
- `CREATE NONCLUSTERED INDEX [IX_Name] ON [Table](Col1, Col2) INCLUDE (Col3) WHERE Col4 IS NOT NULL;`

## References
- [Microsoft Documentation: Index Architecture and Design](https://learn.microsoft.com/en-us/sql/relational-databases/sql-server-index-design-guide)
- [Ola Hallengren's Index Maintenance Script](https://ola.hallengren.com/sql-server-index-and-statistics-maintenance.html)
- [Brent Ozar's Guide to Missing Indexes](https://www.brentozar.com/sql/index-tuning-and-missing-indexes/)
