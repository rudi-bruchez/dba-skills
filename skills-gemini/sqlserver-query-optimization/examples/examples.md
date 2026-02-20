# SQL Server Query Optimization Examples

Practical scenarios and how to optimize slow T-SQL queries.

## Scenario 1: Slow Reports on Large Tables

### Diagnosis
1.  **Top CPU Queries:** Found a query that takes several minutes and has a high CPU cost.
2.  **Execution Plan:** High-cost "Clustered Index Scan" (Table Scan) on the `Orders` table.
3.  **Observation:** The `WHERE` clause is filtering by `OrderDate`.
```sql
SELECT OrderID, CustomerID, OrderDate, TotalAmount
FROM Sales.Orders
WHERE OrderDate >= '2023-01-01' AND OrderDate < '2023-02-01';
```
4.  **Root Cause:** No index exists on the `OrderDate` column.

### Optimization
Create a non-clustered index on `OrderDate`.
```sql
CREATE NONCLUSTERED INDEX IX_Orders_OrderDate
ON Sales.Orders (OrderDate)
INCLUDE (CustomerID, TotalAmount); -- Including extra columns to avoid lookups
```
**Result:** The execution plan now uses an "Index Seek" on `IX_Orders_OrderDate`, significantly reducing the CPU and I/O cost.

## Scenario 2: High "Key Lookup" Costs

### Diagnosis
1.  **Execution Plan:** A query has an "Index Seek" on a non-clustered index, but it's followed by a high-cost "Key Lookup" (95% of the total query cost).
2.  **Query:**
```sql
SELECT CustomerID, AccountBalance
FROM Bank.Accounts
WHERE AccountNumber = '123456';
```
3.  **Index:** An index exists on `AccountNumber`, but it doesn't include `CustomerID` and `AccountBalance`.

### Optimization
Add the requested columns to the index's `INCLUDE` clause.
```sql
CREATE NONCLUSTERED INDEX IX_Accounts_AccountNumber
ON Bank.Accounts (AccountNumber)
INCLUDE (CustomerID, AccountBalance)
WITH (DROP_EXISTING = ON);
```
**Result:** The "Key Lookup" is eliminated. All necessary data is now fetched from the non-clustered index itself (a "Covering Index").

## Scenario 3: Identifying Non-SARGable Queries

### Diagnosis
1.  **Execution Plan:** High-cost "Clustered Index Scan" on the `Users` table.
2.  **Query:**
```sql
SELECT UserID, UserName
FROM dbo.Users
WHERE LOWER(UserName) = 'rudi';
```
3.  **Observation:** An index exists on `UserName`, but it's not being used.

### Optimization
Rewrite the query to be "SARGable" (if using a case-insensitive collation).
```sql
SELECT UserID, UserName
FROM dbo.Users
WHERE UserName = 'rudi'; -- The collation will handle the case-insensitivity
```
If using a case-sensitive collation, add a computed column with an expression index.
```sql
ALTER TABLE Users ADD UserName_Lower AS LOWER(UserName) PERSISTED;
CREATE INDEX IX_Users_UserName_Lower ON Users (UserName_Lower);
```
**Result:** The execution plan now uses an "Index Seek" on `IX_Users_UserName_Lower` or `IX_Users_UserName`.

## Scenario 4: Parameter Sniffing Issues

### Diagnosis
1.  **Observation:** A query performs well for some parameters but poorly for others.
2.  **Action:** Check the cached execution plan and identify if it was optimized for an atypical parameter.

### Optimization
Use the `OPTION (RECOMPILE)` hint to force a new plan for each execution.
```sql
SELECT * FROM Sales.Orders WHERE CustomerID = @CustomerID OPTION (RECOMPILE);
```
Alternatively, use `OPTIMIZE FOR` if there's a "typical" parameter value.
```sql
SELECT * FROM Sales.Orders WHERE CustomerID = @CustomerID OPTION (OPTIMIZE FOR (@CustomerID = 1));
```
**Result:** SQL Server will generate an execution plan specifically tailored for the provided parameter value.
