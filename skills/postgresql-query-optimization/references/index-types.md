# PostgreSQL Index Types Reference

## Index Type Summary

| Index Type | Operators Supported | Best For | Relative Size |
|-----------|-------------------|---------|--------------|
| B-tree | `=`, `<`, `>`, `<=`, `>=`, `BETWEEN`, `LIKE 'prefix%'`, `IS NULL`, `IS NOT NULL` | Most queries | Medium |
| Hash | `=` only | Equality-only lookups | Small |
| GIN | `@>`, `<@`, `&&`, `@@`, `@?` | JSONB, arrays, full-text | Large |
| GiST | Geometric, range overlap, nearest-neighbor | Geometric, range, PostGIS | Medium-Large |
| BRIN | Range summaries per block | Very large append-only tables | Very small |
| SP-GiST | Non-balanced trees | IP addresses, phone numbers | Medium |

---

## B-tree Indexes

The default index type. Balanced tree structure supporting ordered operations.

### When to Use
- Equality and range queries (`=`, `<`, `>`, `BETWEEN`)
- `ORDER BY` and `GROUP BY` on indexed columns
- `LIKE 'prefix%'` (prefix matches only, not `LIKE '%suffix'`)
- Primary keys and unique constraints

### Syntax

```sql
-- Simple index
CREATE INDEX idx_orders_customer ON orders (customer_id);

-- Unique index
CREATE UNIQUE INDEX idx_users_email ON users (email);

-- Composite index (column order matters for queries)
CREATE INDEX idx_orders_customer_date ON orders (customer_id, order_date);

-- Descending sort order
CREATE INDEX idx_orders_date_desc ON orders (created_at DESC NULLS LAST);

-- Case-insensitive string lookup
CREATE INDEX idx_users_lower_email ON users (lower(email));
-- Query must use: WHERE lower(email) = 'foo@example.com'
```

### Composite Index Column Order

The leftmost prefix rule: a composite index on `(a, b, c)` supports queries filtering on:
- `a` alone
- `a, b` together
- `a, b, c` together
- `ORDER BY a`, `ORDER BY a, b`

It does NOT efficiently support queries on `b` alone or `c` alone.

```sql
-- Index on (customer_id, order_date)
-- Supports: WHERE customer_id = 42
-- Supports: WHERE customer_id = 42 AND order_date > '2025-01-01'
-- Supports: WHERE customer_id = 42 ORDER BY order_date
-- Does NOT efficiently support: WHERE order_date > '2025-01-01' (no customer_id filter)
```

### Partial Indexes

Only index rows matching a condition. Smaller, faster to maintain, more selective.

```sql
-- Index only active orders (most queries look for active orders)
CREATE INDEX idx_active_orders ON orders (customer_id)
WHERE status = 'active';

-- Index non-null values only
CREATE INDEX idx_users_email ON users (email)
WHERE email IS NOT NULL;

-- Index recent data only
CREATE INDEX idx_recent_events ON events (user_id, created_at)
WHERE created_at > '2025-01-01';
```

**Rule:** The query's WHERE clause must include the partial index condition (or imply it) for the planner to use the index.

### Covering Indexes (INCLUDE Columns)

```sql
-- Query: SELECT status, total FROM orders WHERE customer_id = 42 ORDER BY order_date
-- Standard index: requires heap fetch for status and total
CREATE INDEX idx_orders_customer ON orders (customer_id, order_date);

-- Covering index: all columns available in index, no heap visit needed
CREATE INDEX idx_orders_covering ON orders (customer_id, order_date)
INCLUDE (status, total);
```

`INCLUDE` columns are stored in the index leaf pages but are not part of the key (not sorted, not used for filtering). They enable Index Only Scans for queries that need those columns.

---

## Hash Indexes

Simple hash table. Only supports equality (`=`). Smaller than B-tree for equality-only lookups.

### When to Use
- Equality-only lookup columns (no range queries, no ORDER BY needed)
- Very large equality-join columns (UUID foreign keys)

```sql
CREATE INDEX idx_sessions_token ON sessions USING HASH (session_token);
```

**Note:** Hash indexes were not WAL-logged in PostgreSQL < 10 (they were lost on crash). Since PostgreSQL 10, they are safe for production use.

---

## GIN (Generalized Inverted Index)

Inverted index mapping element values to the rows that contain them. Efficient for multi-value columns.

### When to Use
- JSONB containment and existence queries
- Array containment (`@>`, `&&`)
- Full-text search (`@@`)
- `pg_trgm` trigram similarity (LIKE '%substring%')

### JSONB Queries

```sql
-- GIN index for containment operator @>
CREATE INDEX idx_events_payload ON events USING GIN (payload);

-- Containment: find events where payload contains {"type": "login"}
SELECT * FROM events WHERE payload @> '{"type": "login"}';

-- Key existence: find events that have an "error" key
SELECT * FROM events WHERE payload ? 'error';

-- Path expression: requires jsonb_path_ops (smaller index, fewer operators)
CREATE INDEX idx_events_pathops ON events USING GIN (payload jsonb_path_ops);
SELECT * FROM events WHERE payload @> '{"user": {"id": 42}}';
```

**GIN opclass options for JSONB:**
- `jsonb_ops` (default): supports `@>`, `?`, `?|`, `?&` — larger index
- `jsonb_path_ops`: supports only `@>` — about 3x smaller, faster builds

### Array Indexes

```sql
-- GIN for array containment
CREATE INDEX idx_articles_tags ON articles USING GIN (tags);

-- Find articles tagged with 'postgres'
SELECT * FROM articles WHERE tags @> ARRAY['postgres'];

-- Find articles with any of these tags
SELECT * FROM articles WHERE tags && ARRAY['postgres', 'performance'];
```

### Full-Text Search

```sql
-- GIN on a tsvector column
ALTER TABLE documents ADD COLUMN tsv tsvector
    GENERATED ALWAYS AS (to_tsvector('english', content)) STORED;
CREATE INDEX idx_documents_tsv ON documents USING GIN (tsv);

-- Search query
SELECT * FROM documents WHERE tsv @@ to_tsquery('english', 'postgres & performance');

-- GIN on expression (no stored column needed)
CREATE INDEX idx_docs_fts ON documents USING GIN (to_tsvector('english', content));
SELECT * FROM documents WHERE to_tsvector('english', content) @@ to_tsquery('postgres');
```

### Trigram Similarity (pg_trgm extension)

```sql
CREATE EXTENSION pg_trgm;

-- GIN trigram index enables fast LIKE '%substring%' queries
CREATE INDEX idx_users_name_trgm ON users USING GIN (name gin_trgm_ops);

-- Now these queries use the index:
SELECT * FROM users WHERE name LIKE '%ohnso%';
SELECT * FROM users WHERE name ILIKE '%OHNSO%';
SELECT * FROM users WHERE name % 'Johnson';  -- similarity search
```

---

## GiST (Generalized Search Tree)

A framework for balanced search trees supporting custom operators. Used for geometric, range, and similarity searches.

### When to Use
- Geometric types (`point`, `circle`, `polygon`, `box`, `line`)
- Range types (`int4range`, `daterange`, `tsrange`, `numrange`)
- Full-text search (alternative to GIN; supports nearest-neighbor)
- PostGIS spatial data

```sql
-- Range type: find overlapping bookings
CREATE INDEX idx_bookings_period ON bookings USING GIST (during);
SELECT * FROM bookings WHERE during && '[2025-06-01, 2025-06-15]'::daterange;

-- Geometric: find points within a circle
CREATE INDEX idx_locations_geo ON locations USING GIST (coordinates);
SELECT * FROM locations WHERE coordinates <@ circle '((0,0), 100)';

-- Nearest-neighbor search (only GiST, not GIN)
SELECT * FROM locations ORDER BY coordinates <-> point '(10,20)' LIMIT 5;
```

---

## BRIN (Block Range INdex)

Stores min/max values for a range of heap blocks. Very small, very fast to build, but imprecise — good only when values are physically correlated with their location on disk.

### When to Use
- Very large tables (hundreds of GB) with append-only or sequential insert patterns
- Time-series data where `created_at` increases monotonically
- Tables that are too large for a B-tree index

```sql
-- Time-series table where created_at is always increasing
CREATE INDEX idx_events_created_brin ON events USING BRIN (created_at);

-- Custom block range size (default: 128 pages)
CREATE INDEX idx_events_brin ON events USING BRIN (created_at) WITH (pages_per_range = 32);
```

**Limitation:** BRIN only works when the column is physically correlated with insertion order. For a column where rows are scattered (e.g., `updated_at` which changes randomly), BRIN provides no benefit. Check correlation:

```sql
SELECT attname, correlation
FROM pg_stats
WHERE tablename = 'events' AND attname = 'created_at';
-- correlation close to 1.0 or -1.0 = BRIN will work well
-- correlation close to 0 = BRIN will be nearly useless
```

---

## Index Maintenance

### Check Index Health

```sql
-- Index usage statistics
SELECT schemaname, tablename, indexname, idx_scan, idx_tup_read, idx_tup_fetch,
       pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC
LIMIT 20;

-- Find unused indexes (idx_scan = 0 since last stats reset)
SELECT schemaname, tablename, indexname, idx_scan,
       pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND schemaname NOT IN ('pg_catalog', 'pg_toast')
ORDER BY pg_relation_size(indexrelid) DESC;

-- Find duplicate indexes (same table and key columns)
SELECT indrelid::regclass AS table_name,
       array_agg(indexrelid::regclass) AS indexes,
       array_agg(pg_get_indexdef(indexrelid)) AS definitions
FROM pg_index
GROUP BY indrelid, indkey
HAVING count(*) > 1;
```

### Rebuild Indexes

```sql
-- Rebuild a single index online (no write lock)
REINDEX INDEX CONCURRENTLY idx_orders_customer;

-- Rebuild all indexes on a table online
REINDEX TABLE CONCURRENTLY orders;

-- Rebuild all indexes in a schema
REINDEX SCHEMA CONCURRENTLY myschema;
```

### Drop an Index Safely

```sql
-- Drop index concurrently (does not block reads or writes)
DROP INDEX CONCURRENTLY idx_old_unused;
```
