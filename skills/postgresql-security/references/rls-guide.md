# Row-Level Security (RLS) Guide

## How RLS Works

Row-Level Security adds WHERE clause conditions to every query against a table. When RLS is enabled:
- Without a matching policy: the default behavior is to deny all rows (no rows returned for SELECT, all INSERT/UPDATE/DELETE blocked)
- With `BYPASSRLS` role attribute or table owner: policies are bypassed by default
- With `FORCE ROW LEVEL SECURITY`: table owner is also restricted

---

## Enabling RLS

```sql
-- Enable RLS on a table
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Verify RLS status
SELECT relname, relrowsecurity, relforcerowsecurity
FROM pg_class
WHERE relname = 'orders';
-- relrowsecurity = true  (RLS is on)
-- relforcerowsecurity = false  (table owner bypasses)

-- Force RLS even for table owner
ALTER TABLE orders FORCE ROW LEVEL SECURITY;

-- Disable RLS (removes all filtering)
ALTER TABLE orders DISABLE ROW LEVEL SECURITY;
```

---

## Policy Syntax

```sql
CREATE POLICY policy_name ON table_name
    [AS { PERMISSIVE | RESTRICTIVE }]
    [FOR { ALL | SELECT | INSERT | UPDATE | DELETE }]
    [TO { role_name | PUBLIC | CURRENT_USER | SESSION_USER } [, ...]]
    [USING ( using_expression )]
    [WITH CHECK ( check_expression )];
```

| Clause | When Used | What It Does |
|--------|-----------|-------------|
| `USING` | SELECT, UPDATE, DELETE | Filters visible rows (added to WHERE clause) |
| `WITH CHECK` | INSERT, UPDATE | Validates new row values (prevents writing rows that would not be visible) |
| `PERMISSIVE` | Default | Policies are OR'd together (any matching policy allows access) |
| `RESTRICTIVE` | Explicit | This policy is AND'd with all permissive policies (must also pass this) |

---

## Common RLS Patterns

### Pattern 1: User Isolation (Each User Sees Own Rows)

```sql
-- Application sets current user ID before queries:
-- SET app.current_user_id = '42';

ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

CREATE POLICY user_isolation ON orders
    USING (user_id = current_setting('app.current_user_id', true)::int);
-- true = return null if setting is not set (instead of error)
```

### Pattern 2: Tenant Isolation (Multi-Tenant SaaS)

```sql
-- Application sets tenant ID on each connection:
-- SET app.tenant_id = '100';

ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders FORCE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON orders
    USING (tenant_id = current_setting('app.tenant_id', true)::int)
    WITH CHECK (tenant_id = current_setting('app.tenant_id', true)::int);
```

### Pattern 3: Role-Based Access

```sql
-- Managers can see all rows; employees see only their department
ALTER TABLE hr_records ENABLE ROW LEVEL SECURITY;

CREATE POLICY manager_full_access ON hr_records
    FOR ALL
    TO manager_role
    USING (true);  -- Managers see all rows

CREATE POLICY employee_own_dept ON hr_records
    FOR SELECT
    TO employee_role
    USING (department_id = current_setting('app.department_id', true)::int);
```

### Pattern 4: Admin Bypass Policy

```sql
-- Admins bypass row filtering; regular users see filtered data
CREATE POLICY admin_bypass ON orders
    USING (pg_has_role(current_user, 'admin_role', 'member'));

CREATE POLICY user_own_rows ON orders
    USING (user_id = current_setting('app.current_user_id', true)::int);

-- Both policies are PERMISSIVE: admin sees all; user sees own rows
-- (If user is also admin, they see all rows)
```

### Pattern 5: Security-Definer Function for Context

Using a security-definer function prevents direct inspection of the session variable expression:

```sql
-- Function runs with definer's privileges (safe from session tampering)
CREATE OR REPLACE FUNCTION current_tenant_id() RETURNS int
    LANGUAGE sql
    SECURITY DEFINER
    STABLE
    AS $$
        SELECT NULLIF(current_setting('app.tenant_id', true), '')::int
    $$;

-- Policy uses the function
CREATE POLICY tenant_isolation ON orders
    USING (tenant_id = current_tenant_id())
    WITH CHECK (tenant_id = current_tenant_id());
```

### Pattern 6: Separate Policies per Operation

```sql
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Different rules for SELECT vs modification
CREATE POLICY orders_select ON orders
    FOR SELECT
    USING (customer_id = current_setting('app.current_user_id', true)::int
           OR pg_has_role(current_user, 'support_role', 'member'));

CREATE POLICY orders_insert ON orders
    FOR INSERT
    WITH CHECK (customer_id = current_setting('app.current_user_id', true)::int);

CREATE POLICY orders_update ON orders
    FOR UPDATE
    USING (customer_id = current_setting('app.current_user_id', true)::int)
    WITH CHECK (customer_id = current_setting('app.current_user_id', true)::int);

-- No DELETE policy: prevents all deletes except for users with BYPASSRLS
```

### Pattern 7: Restrictive Policy (Mandatory Filter)

A restrictive policy is AND'd with all permissive policies — all conditions must pass.

```sql
-- Restrictive: always filter deleted records regardless of other policies
CREATE POLICY no_deleted_rows ON orders
    AS RESTRICTIVE
    USING (deleted_at IS NULL);

-- Permissive: user sees own orders (but the restrictive policy also applies)
CREATE POLICY user_own_orders ON orders
    USING (user_id = current_setting('app.current_user_id', true)::int);

-- Result: user sees their own orders WHERE deleted_at IS NULL
```

---

## Setting Application Context

```sql
-- Set session variable (persists for the session)
SET app.tenant_id = '42';
SET app.current_user_id = '100';

-- Set within a transaction (reverts on COMMIT or ROLLBACK)
BEGIN;
SET LOCAL app.tenant_id = '42';
-- All queries in this transaction see tenant 42's rows
COMMIT;

-- Set and immediately use (in applications using connection pools)
-- Transaction-level pooling (PgBouncer transaction mode):
BEGIN;
SET LOCAL app.tenant_id = :tenant_id;
SELECT * FROM orders WHERE status = 'pending';  -- Filtered by RLS
COMMIT;
-- After COMMIT, the variable is cleared
```

---

## Viewing RLS Policies

```sql
-- List all policies
SELECT schemaname, tablename, policyname,
       permissive, roles, cmd, qual, with_check
FROM pg_policies
WHERE schemaname = 'public'
ORDER BY tablename, policyname;

-- Check if RLS is enabled
SELECT relname, relrowsecurity, relforcerowsecurity
FROM pg_class
WHERE relnamespace = 'public'::regnamespace
  AND relkind = 'r'
ORDER BY relname;
```

---

## Managing Policies

```sql
-- Alter a policy (change the USING expression)
ALTER POLICY tenant_isolation ON orders
    USING (tenant_id = current_setting('app.tenant_id', true)::int
           AND active = true);

-- Enable/disable a policy without dropping it
ALTER POLICY tenant_isolation ON orders ENABLE;
ALTER POLICY tenant_isolation ON orders DISABLE;

-- Drop a policy
DROP POLICY tenant_isolation ON orders;

-- Drop all policies and disable RLS
DROP POLICY IF EXISTS tenant_isolation ON orders;
ALTER TABLE orders DISABLE ROW LEVEL SECURITY;
```

---

## Bypass RLS

```sql
-- Role with BYPASSRLS bypasses all policies (like superuser for RLS)
CREATE ROLE etl_admin WITH LOGIN BYPASSRLS PASSWORD 'password';

-- Grant BYPASSRLS to an existing role
ALTER ROLE existing_role BYPASSRLS;

-- Check who has BYPASSRLS
SELECT rolname FROM pg_roles WHERE rolbypassrls = true;
```

---

## Performance Considerations

1. RLS policies add WHERE conditions to every query. Ensure the policy columns have indexes:
   ```sql
   -- If policy uses tenant_id, index it
   CREATE INDEX ON orders (tenant_id);
   ```

2. Use `SECURITY DEFINER` functions for complex policy logic — they can be optimized by the planner.

3. Test policy performance with `EXPLAIN (ANALYZE, BUFFERS)`:
   ```sql
   SET app.tenant_id = '42';
   EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE status = 'active';
   -- Plan should show: Filter: ((tenant_id = 42) AND (status = 'active'))
   -- Verify index is used for tenant_id
   ```

4. For high-throughput scenarios, use connection pooling where each pool serves a single tenant, so the connection's session variable is set once on connect.
