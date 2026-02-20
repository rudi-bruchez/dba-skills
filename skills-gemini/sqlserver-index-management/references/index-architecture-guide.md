# Index Architecture Guide

A guide to the physical and logical structure of SQL Server indexes.

## 1. B-tree Structure
Most SQL Server indexes are stored in a B-tree structure, which allows for efficient searching and range scans.
- **Root Level:** The entry point of the index.
- **Intermediate Levels:** Contain pointers to lower levels or leaf pages.
- **Leaf Level:** Contains the actual data (for clustered indexes) or pointers to data (for non-clustered indexes).

## 2. Clustered vs. Non-Clustered Indexes
- **Clustered Index:** Defines the physical order of data in the table. Each table can have only one.
- **Non-Clustered Index:** A separate structure that points to the data rows. A table can have many.

## 3. Fill Factor
The percentage of space to fill on each leaf-level page during index creation or rebuild.
- **Target:** 100% (or 0) for read-only tables; 80-90% for tables with frequent inserts to avoid page splits.

## 4. Page Splits
Occur when a page is full and a new row must be inserted, forcing SQL Server to move some data to a new page. This causes fragmentation and reduces performance.

## 5. Statistics
The Query Optimizer uses statistics (histograms, density vectors) to estimate the number of rows and choose the best execution plan. Rebuilding an index updates statistics with `FULLSCAN`.

## References
- [Microsoft Documentation: Index Architecture and Design](https://learn.microsoft.com/en-us/sql/relational-databases/sql-server-index-design-guide)
- [SQLskills: Index Internals and Performance](https://www.sqlskills.com/training/online-training/index-internals-and-performance/)
