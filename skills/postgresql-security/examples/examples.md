# PostgreSQL Security Examples

## Scenario 1: Full Privilege Audit

**Situation:** A compliance audit requires documenting all PostgreSQL access controls. The DBA must inventory all roles, their privileges, and flag any security concerns.

**Step 1 — Find all superusers:**
```sql
SELECT rolname, rolsuper, rolcreaterole, rolcreatedb, rolreplication
FROM pg_roles
WHERE rolsuper = true;
```
Result: `postgres` (expected) and `app_admin` (unexpected — application role should not be superuser).

**Finding:** `app_admin` has superuser access. This is a critical risk — a compromised application account gains full database control.

**Fix:**
```sql
ALTER ROLE app_admin NOSUPERUSER NOCREATEROLE NOCREATEDB;
```

**Step 2 — Review all login roles:**
```sql
SELECT rolname, rolsuper, rolcreaterole, rolcreatedb,
       rolconnlimit, rolvaliduntil
FROM pg_roles
WHERE rolcanlogin = true
ORDER BY rolname;
```
Result shows:
- `app_service`: `rolvaliduntil = NULL` (no expiry), `rolconnlimit = -1` (unlimited connections)
- `old_etl_job`: `rolvaliduntil = '2023-01-01'` (expired but still exists)

**Fixes:**
```sql
-- Add password expiry and connection limit
ALTER ROLE app_service
    PASSWORD 'new_strong_password'
    VALID UNTIL '2026-12-31'
    CONNECTION LIMIT 50;

-- Remove unused expired role
REASSIGN OWNED BY old_etl_job TO postgres;
DROP OWNED BY old_etl_job;
DROP ROLE old_etl_job;
```

**Step 3 — Audit table access:**
```sql
SELECT table_schema, table_name, grantee, privilege_type
FROM information_schema.role_table_grants
WHERE table_schema = 'public'
ORDER BY table_name, grantee;
```
Result: `app_service` has DELETE privilege on `orders` — this application account should never delete orders.

**Fix:**
```sql
REVOKE DELETE ON public.orders FROM app_service;
```

**Step 4 — Check authentication methods:**
```sql
SELECT line_number, type, database, user_name, address, auth_method
FROM pg_hba_file_rules
WHERE auth_method IN ('trust', 'password', 'md5');
```
Result: Line 8 has `md5` for `host all all 10.0.0.0/8 md5`.

**Fix (edit pg_hba.conf):**
```
# Change from:
host  all  all  10.0.0.0/8  md5
# To:
host  all  all  10.0.0.0/8  scram-sha-256
```

```sql
-- Force existing passwords to be rehashed on next change
ALTER SYSTEM SET password_encryption = 'scram-sha-256';
SELECT pg_reload_conf();

-- Have each user reset their password to trigger rehash
ALTER ROLE app_service PASSWORD 'same_or_new_password';
ALTER ROLE app_readonly PASSWORD 'same_or_new_password';
```

---

## Scenario 2: Fixing Public Schema Access

**Situation:** A security scan flags that the `public` schema allows CREATE privilege for all users. Any authenticated user can install functions or objects into the public schema (table injection risk).

**Step 1 — Verify the issue:**
```sql
SELECT nspname, nspacl
FROM pg_namespace
WHERE nspname = 'public';
-- nspacl might show: {postgres=UC/postgres,=UC/postgres}
-- The "=UC/postgres" part means PUBLIC has USAGE and CREATE
```

```sql
-- Alternative check using psql \dn+ output equivalent
SELECT nspname,
       has_schema_privilege('PUBLIC', nspname, 'CREATE') AS public_can_create
FROM pg_namespace
WHERE nspname = 'public';
```

**Step 2 — Revoke CREATE from public schema:**
```sql
REVOKE CREATE ON SCHEMA public FROM PUBLIC;

-- Also lock down the database itself
REVOKE ALL ON DATABASE mydb FROM PUBLIC;

-- Verify the change
SELECT has_schema_privilege('PUBLIC', 'public', 'CREATE') AS public_can_create;
-- Should return: false
```

**Step 3 — Grant necessary access back to application roles:**
```sql
-- Application roles need USAGE on public schema (not CREATE)
GRANT USAGE ON SCHEMA public TO app_service, app_readonly, app_readwrite;

-- Allow connect to specific roles only
GRANT CONNECT ON DATABASE mydb TO app_service, app_readonly, app_readwrite;
```

**Step 4 — Verify application still works:**
```sql
-- Test as app_service
SET ROLE app_service;
SELECT count(*) FROM orders;   -- Should work
CREATE TABLE test_injection (id int);  -- Should FAIL with "permission denied"
RESET ROLE;
```

**Step 5 — Set a secure search_path for application roles:**
```sql
-- Prevent schema injection via search_path manipulation
ALTER ROLE app_service SET search_path = myapp, public;
ALTER ROLE app_readonly SET search_path = myapp, public;
```

---

## Scenario 3: Implementing Row-Level Security for Multi-Tenant SaaS

**Situation:** A SaaS application stores all customers' data in a single PostgreSQL database. Currently, all queries return data for all tenants. Need to implement tenant isolation at the database level.

**Schema:**
```sql
-- Existing tables (simplified)
CREATE TABLE orders (
    id bigserial PRIMARY KEY,
    tenant_id int NOT NULL,
    customer_id int NOT NULL,
    total numeric(10,2),
    status text,
    created_at timestamptz DEFAULT now()
);

CREATE INDEX idx_orders_tenant ON orders (tenant_id);
```

**Step 1 — Enable RLS on tenant tables:**
```sql
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders FORCE ROW LEVEL SECURITY;
-- FORCE ensures even the table owner (app_admin) is filtered
```

**Step 2 — Create the isolation function:**
```sql
CREATE OR REPLACE FUNCTION current_tenant_id()
RETURNS int LANGUAGE sql SECURITY DEFINER STABLE AS $$
    SELECT NULLIF(current_setting('app.tenant_id', true), '')::int
$$;

-- Grant execute to application roles
GRANT EXECUTE ON FUNCTION current_tenant_id() TO app_service;
```

**Step 3 — Create tenant isolation policies:**
```sql
-- Main isolation policy: tenants only see their own data
CREATE POLICY tenant_isolation ON orders
    FOR ALL
    USING (tenant_id = current_tenant_id())
    WITH CHECK (tenant_id = current_tenant_id());

-- Support staff can see all tenants (permissive, any matching policy works)
CREATE POLICY support_access ON orders
    FOR SELECT
    TO support_role
    USING (true);
```

**Step 4 — Update application connection handling:**
```python
# Python psycopg2 example: set tenant context at start of each request
def get_connection(tenant_id):
    conn = pool.getconn()
    with conn.cursor() as cur:
        cur.execute("SET LOCAL app.tenant_id = %s", (str(tenant_id),))
    return conn

# Transaction pooling (PgBouncer) example: set per-transaction
def execute_for_tenant(tenant_id, query, params):
    with connection.cursor() as cur:
        cur.execute("BEGIN")
        cur.execute("SET LOCAL app.tenant_id = %s", (str(tenant_id),))
        cur.execute(query, params)
        results = cur.fetchall()
        cur.execute("COMMIT")
        return results
```

**Step 5 — Test the isolation:**
```sql
-- Test as tenant 1
SET app.tenant_id = '1';
SELECT count(*) FROM orders;
-- Returns only tenant 1's orders

SET app.tenant_id = '2';
SELECT count(*) FROM orders;
-- Returns only tenant 2's orders

-- Attempt to insert a row for a different tenant
SET app.tenant_id = '1';
INSERT INTO orders (tenant_id, customer_id, total) VALUES (2, 100, 99.99);
-- ERROR: new row violates row-level security policy for table "orders"
```

---

## Scenario 4: Configuring SSL for Production

**Situation:** A new PostgreSQL 16 server has been set up but SSL is not yet enabled. The application connects over an internal network but must encrypt connections for compliance.

**Step 1 — Generate server certificate (using Let's Encrypt or internal CA):**
```bash
# For a self-signed certificate (development/internal only):
openssl genrsa -out /var/lib/postgresql/server.key 2048
chmod 600 /var/lib/postgresql/server.key
chown postgres:postgres /var/lib/postgresql/server.key

openssl req -new -x509 -days 365 \
    -key /var/lib/postgresql/server.key \
    -out /var/lib/postgresql/server.crt \
    -subj "/CN=db.internal.example.com"

chown postgres:postgres /var/lib/postgresql/server.crt
```

**Step 2 — Enable SSL in postgresql.conf:**
```ini
ssl = on
ssl_cert_file = '/var/lib/postgresql/server.crt'
ssl_key_file = '/var/lib/postgresql/server.key'
ssl_min_protocol_version = 'TLSv1.2'
```

```bash
# Reload or restart
systemctl reload postgresql   # If ssl was already on
# systemctl restart postgresql  # If ssl was off (requires restart)
```

**Step 3 — Update pg_hba.conf to require SSL:**
```
# Change host to hostssl for application connections
# Before:
host  all  app_service  10.0.0.0/8  scram-sha-256

# After: SSL required
hostssl  all  app_service  10.0.0.0/8  scram-sha-256
```

```sql
SELECT pg_reload_conf();
```

**Step 4 — Verify SSL is working:**
```sql
SELECT ssl, version, cipher, bits, client_addr
FROM pg_stat_ssl
JOIN pg_stat_activity USING (pid)
WHERE usename = 'app_service';
```
Expected: `ssl = true`, `version = TLSv1.3` (or TLSv1.2), `cipher` shows a strong cipher.

**Step 5 — Update application connection strings:**
```bash
# Python psycopg2
conn = psycopg2.connect(
    "host=db.internal.example.com dbname=myapp user=app_service "
    "sslmode=require sslrootcert=/etc/ssl/certs/internal-ca.crt"
)

# Connection string
postgresql://app_service@db.internal.example.com/myapp?sslmode=verify-full&sslrootcert=/etc/ssl/certs/internal-ca.crt
```

**Step 6 — Find any remaining unencrypted connections:**
```sql
SELECT usename, application_name, client_addr, ssl
FROM pg_stat_ssl
JOIN pg_stat_activity USING (pid)
WHERE client_addr IS NOT NULL   -- Exclude local unix socket connections
ORDER BY ssl, usename;
```
