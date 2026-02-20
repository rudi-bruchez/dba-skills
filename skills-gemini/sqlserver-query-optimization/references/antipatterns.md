# Query Tuning Anti-patterns

Common T-SQL coding practices that cause performance degradation and how to fix them.

## 1. Non-SARGable Queries

A "Search ARGumentable" (SARGable) query is one where the SQL Server Query Optimizer can effectively use an index. Using functions or operations on an indexed column in the `WHERE` clause prevents this.

**Anti-Pattern:**
```sql
SELECT * FROM Sales.Orders WHERE YEAR(OrderDate) = 2023;
```
**Fix:**
```sql
SELECT * FROM Sales.Orders WHERE OrderDate >= '2023-01-01' AND OrderDate < '2024-01-01';
```

**Anti-Pattern:**
```sql
SELECT * FROM Users WHERE LOWER(EmailAddress) = 'rudi@example.com';
```
**Fix:**
Use a case-insensitive collation or a computed column with an expression index.
```sql
ALTER TABLE Users ADD Email_Lower AS LOWER(EmailAddress) PERSISTED;
CREATE INDEX IX_Users_Email_Lower ON Users (Email_Lower);
```

## 2. Using `SELECT *`

Fetching all columns from a table when only a few are needed increases I/O, memory usage, and prevents the use of covering indexes.

**Anti-Pattern:**
```sql
SELECT * FROM Sales.Orders WHERE CustomerID = 123;
```
**Fix:**
```sql
SELECT OrderDate, TotalAmount FROM Sales.Orders WHERE CustomerID = 123;
```

## 3. Implicit Conversions

Implicit conversions occur when the data type of a column doesn't match the data type of the value being compared, forcing SQL Server to convert the column values for every row, which prevents index usage.

**Anti-Pattern:**
```sql
-- Assume AccountNumber is a VARCHAR(20)
SELECT * FROM Accounts WHERE AccountNumber = 12345; -- Integer vs. VARCHAR
```
**Fix:**
```sql
SELECT * FROM Accounts WHERE AccountNumber = '12345'; -- VARCHAR vs. VARCHAR
```

## 4. Scalar User-Defined Functions (UDFs)

Scalar UDFs are executed row-by-row, which can be extremely slow for large result sets and prevent parallelism.

**Anti-Pattern:**
```sql
SELECT dbo.fn_GetTotal(OrderID) FROM Sales.Orders;
```
**Fix:**
Rewrite the function logic as a "Cross Apply" or an "Inline Table-Valued Function" (iTVF).

## 5. Correlated Subqueries

Correlated subqueries are executed for every row in the outer query.

**Anti-Pattern:**
```sql
SELECT CustomerName, (SELECT TOP 1 OrderDate FROM Orders WHERE Orders.CustomerID = Customers.CustomerID ORDER BY OrderDate DESC) AS LastOrderDate
FROM Customers;
```
**Fix:**
```sql
SELECT c.CustomerName, o.LastOrderDate
FROM Customers AS c
OUTER APPLY (SELECT TOP 1 OrderDate AS LastOrderDate FROM Orders WHERE Orders.CustomerID = c.CustomerID ORDER BY OrderDate DESC) AS o;
```

## 2024-2025 Considerations
- **SQL Server 2019+:** Scalar UDF Inlining can automatically optimize many scalar functions.
- **SQL Server 2022+:** Parameter Sensitive Plan (PSP) optimization helps mitigate issues with parameter sniffing.
