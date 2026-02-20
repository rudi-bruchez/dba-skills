# Index Types

PostgreSQL provides several different index types, each optimized for specific data types and query patterns.

## 1. B-tree Index (Default)
Optimized for sorted and range-based data.
- **Best Use Case:** Most common data types (`INT`, `VARCHAR`, `TIMESTAMP`).
- **Supports:** `=`, `>`, `>=`, `<`, `<=`, `BETWEEN`, `IN`, `IS NULL`.

## 2. GIN (Generalized Inverted Index)
Optimized for multi-value columns where multiple values are stored in a single field.
- **Best Use Case:** `JSONB` columns, arrays, full-text search.
- **Supports:** `contains (@>)`, `exists (?)`, `overlap (&&)`.

## 3. GiST (Generalized Search Tree)
Optimized for complex search conditions and specialized data types.
- **Best Use Case:** Geometric data, range types, full-text search.
- **Supports:** `@>`, `&&`, `|&|`, `&<`, `&>`.

## 4. BRIN (Block Range Index)
Extremely compact index for very large tables where data is physically ordered by a column.
- **Best Use Case:** Log files, time-series data, tables with billions of rows.
- **Supports:** `>`, `>=`, `<`, `<=`, `BETWEEN`.

## 5. Hash Index
Optimized for simple equality comparisons.
- **Best Use Case:** Large `VARCHAR` columns where only `=` comparisons are needed.
- **Supports:** `=`. (Note: B-tree is usually better).

## 6. SP-GiST (Space-Partitioned GiST)
Optimized for non-balanced data structures.
- **Best Use Case:** Quadtries, k-d trees, radix trees (e.g., IP addresses).

## Indexing Strategies

1.  **Partial Indexes:** Index only a subset of rows based on a `WHERE` clause.
    ```sql
    CREATE INDEX idx_pending_orders ON orders (customer_id) WHERE status = 'pending';
    ```
2.  **Expression Indexes:** Index the result of a function.
    ```sql
    CREATE INDEX idx_users_email_lower ON users (LOWER(email));
    ```
3.  **Covering Indexes (via `INCLUDE`):** Include extra columns in the index to enable "Index Only Scans."
    ```sql
    CREATE INDEX idx_orders_customer_id ON orders (customer_id) INCLUDE (order_date, total_amount);
    ```

## References
- [PostgreSQL Documentation: Index Types](https://www.postgresql.org/docs/current/indexes-types.html)
- [pganalyze: Index Advisor](https://pganalyze.com/index-advisor)
