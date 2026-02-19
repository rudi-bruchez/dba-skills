---
name: postgresql-security
description: Audits and hardens PostgreSQL security including role management, pg_hba.conf authentication, SSL/TLS configuration, row-level security policies, and audit logging. Use when performing security reviews, setting up new instances, responding to security incidents, or preparing for compliance audits.
metadata:
  author: dba-skills
  version: "1.0"
  platform: PostgreSQL 13+
---

# PostgreSQL Security

## Security Audit Workflow

Run these steps to assess the current security posture of a PostgreSQL instance:

1. **Superusers** — identify all superuser accounts and verify only essential ones exist
2. **Login roles** — review all roles with login capability for least privilege
3. **Authentication** — audit `pg_hba.conf` for dangerous methods (`trust`, `password`, `md5`)
4. **SSL status** — verify SSL is enabled and connections are encrypted
5. **Public schema** — check if `CREATE` privilege on the public schema is revoked
6. **PUBLIC database access** — verify PUBLIC has only `CONNECT`, not `CREATE`
7. **Row-level security** — verify multi-tenant tables have RLS enabled
8. **Audit logging** — confirm `log_connections`, `log_statement` are appropriate

---

## Step 1 — Role and Privilege Review

### Find All Superuser Accounts

```sql
SELECT rolname, rolsuper, rolcreaterole, rolcreatedb, rolreplication, rolcanlogin
FROM pg_roles
WHERE rolsuper = true
ORDER BY rolname;
```

**Expected:** Only the `postgres` system role should be a superuser. Application accounts must never have `rolsuper = true`.

### Review All Login Roles

```sql
SELECT rolname,
       rolsuper,
       rolcreaterole,
       rolcreatedb,
       rolreplication,
       rolconnlimit,
       rolvaliduntil
FROM pg_roles
WHERE rolcanlogin = true
ORDER BY rolname;
```

**Look for:**
- `rolsuper = true` on any non-system role (critical risk)
- `rolcreaterole = true` — can create/modify roles, effectively privilege escalation
- `rolvaliduntil IS NULL` — no password expiry
- `rolconnlimit = -1` — unlimited connections (use `CONNECTION LIMIT` to restrict)

### Check Role Memberships

```sql
SELECT r.rolname AS member, g.rolname AS member_of
FROM pg_auth_members m
JOIN pg_roles r ON r.oid = m.member
JOIN pg_roles g ON g.oid = m.roleid
ORDER BY g.rolname, r.rolname;
```

### Audit Table Privileges for a Specific Role

```sql
SELECT table_schema, table_name, privilege_type
FROM information_schema.role_table_grants
WHERE grantee = 'app_user'
ORDER BY table_schema, table_name, privilege_type;
```

---

## Step 2 — Authentication (pg_hba.conf)

### View Current Authentication Rules

```sql
-- PostgreSQL 10+
SELECT line_number, type, database, user_name, address, auth_method, options
FROM pg_hba_file_rules
ORDER BY line_number;
```

### Identify Dangerous Authentication Methods

```sql
SELECT line_number, type, database, user_name, address, auth_method
FROM pg_hba_file_rules
WHERE auth_method IN ('trust', 'password', 'md5')
ORDER BY line_number;
```

**Risk by method:**
| Method | Risk | Action |
|--------|------|--------|
| `trust` | Critical — no password | Replace with `scram-sha-256`; allow only for local unix socket + `postgres` role |
| `password` | High — cleartext password | Replace with `scram-sha-256` immediately |
| `md5` | Medium — weak hash | Replace with `scram-sha-256`; vulnerable to replay attacks |
| `scram-sha-256` | Low — strong | Preferred for all new setups |
| `peer` | Low — OS-validated | Good for local Unix socket connections |
| `cert` | Very low — certificate auth | Best for service accounts |

### Recommended pg_hba.conf

```
# TYPE  DATABASE        USER            ADDRESS          METHOD
# Local Unix socket
local   all             postgres                         peer
local   all             all                              scram-sha-256

# IPv4 loopback
host    all             all             127.0.0.1/32     scram-sha-256

# IPv6 loopback
host    all             all             ::1/128          scram-sha-256

# Replication (specific subnet only)
host    replication     replicator      10.0.1.0/24      scram-sha-256

# Application connections (require SSL, specific network)
hostssl all             app_user        10.0.0.0/8       scram-sha-256

# Deny everything else
host    all             all             0.0.0.0/0        reject
```

### Reload pg_hba.conf After Changes

```sql
SELECT pg_reload_conf();
-- Or from OS: pg_ctl reload / systemctl reload postgresql
```

---

## Step 3 — SSL/TLS Configuration

### Check SSL Status

```sql
-- Is SSL enabled?
SHOW ssl;

-- View SSL connections
SELECT pid, usename, application_name, client_addr, ssl, version, cipher, bits
FROM pg_stat_ssl
JOIN pg_stat_activity USING (pid)
WHERE ssl = true;

-- Find unencrypted connections
SELECT pid, usename, client_addr, application_name
FROM pg_stat_ssl
JOIN pg_stat_activity USING (pid)
WHERE ssl = false
  AND client_addr IS NOT NULL;  -- Excludes local unix socket connections
```

### Enable SSL in postgresql.conf

```ini
ssl = on
ssl_cert_file = 'server.crt'
ssl_key_file = 'server.key'
ssl_ca_file = 'root.crt'
ssl_min_protocol_version = 'TLSv1.2'   # Minimum; prefer TLSv1.3
ssl_ciphers = 'HIGH:MEDIUM:+3DES:!aNULL'
```

### Force SSL for Specific Connections in pg_hba.conf

```
# 'hostssl' requires SSL; 'hostnossl' rejects SSL connections
hostssl  all  app_user  10.0.0.0/8  scram-sha-256
```

---

## Step 4 — Public Schema and Database Defaults

PostgreSQL has historically granted `CREATE` on the `public` schema to all users. This is a privilege escalation risk.

### Check Current Public Schema Grants

```sql
SELECT nspname,
       nspacl
FROM pg_namespace
WHERE nspname = 'public';
```

Look for `=UC/postgres` or `=C/postgres` which means PUBLIC has CREATE.

### Revoke Dangerous Defaults

```sql
-- Revoke CREATE from public schema (PostgreSQL 15+ does this by default)
REVOKE CREATE ON SCHEMA public FROM PUBLIC;

-- Revoke CREATE DATABASE from PUBLIC on the database
REVOKE ALL ON DATABASE mydb FROM PUBLIC;

-- Grant back only what is needed:
GRANT CONNECT ON DATABASE mydb TO app_user, readonly_user;
GRANT USAGE ON SCHEMA public TO app_user, readonly_user;
```

---

## Step 5 — Row-Level Security (RLS)

RLS restricts which rows a user can see or modify. Essential for multi-tenant applications.

### Enable and Verify RLS on a Table

```sql
-- Enable RLS
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Verify RLS is enabled
SELECT relname, relrowsecurity, relforcerowsecurity
FROM pg_class
WHERE relname = 'orders';
-- relrowsecurity = true means RLS is on
-- relforcerowsecurity = false means table owner bypasses RLS (default)

-- View existing policies
SELECT tablename, policyname, permissive, roles, cmd, qual
FROM pg_policies
WHERE schemaname = 'public';
```

### Create a Tenant Isolation Policy

```sql
-- Application sets session variable on each connection:
-- SET app.tenant_id = '42';

CREATE POLICY tenant_isolation ON orders
    USING (tenant_id = current_setting('app.tenant_id', true)::int);

-- Allow admins to bypass
CREATE POLICY admin_bypass ON orders
    USING (pg_has_role(current_user, 'admin_role', 'member'));
```

---

## Step 6 — Key Security Hardening Settings

Add to `postgresql.conf`:

```ini
# Audit logging
log_connections = on             # Log all new connections
log_disconnections = on          # Log disconnections with session duration
log_statement = 'ddl'            # Log all DDL; use 'mod' for INSERT/UPDATE/DELETE too
log_min_duration_statement = 1000 # Log queries > 1 second
log_lock_waits = on              # Log lock wait events
deadlock_timeout = 1000          # Deadlock detection timeout (ms)

# Session timeouts (prevent runaway/forgotten sessions)
idle_in_transaction_session_timeout = 300000  # 5 minutes (ms)
statement_timeout = 0            # 0 = no global timeout; set per role

# Lock the search_path to prevent schema injection attacks
# Set per role or in postgresql.conf:
# search_path = '"$user", public'
```

**For compliance auditing, install pgaudit:**
```ini
shared_preload_libraries = 'pgaudit'
pgaudit.log = 'write, ddl'     # Log INSERT/UPDATE/DELETE and all DDL
pgaudit.log_catalog = off       # Don't audit catalog queries (too verbose)
pgaudit.log_relation = on       # Log relation name in each statement
pgaudit.log_parameter = on      # Log bind parameters
```

```sql
CREATE EXTENSION pgaudit;
```

---

## Security Hardening Checklist

| Check | Query/Command | Expected | Fix |
|-------|--------------|---------|-----|
| No superusers except postgres | `SELECT rolname FROM pg_roles WHERE rolsuper` | Only `postgres` | `ALTER ROLE username NOSUPERUSER` |
| No `trust` auth in pg_hba | `pg_hba_file_rules WHERE auth_method = 'trust'` | 0 rows (or local-only) | Change to `scram-sha-256` |
| SSL enabled | `SHOW ssl` | `on` | Set `ssl = on` in postgresql.conf |
| TLS 1.2 minimum | `SHOW ssl_min_protocol_version` | `TLSv1.2` or `TLSv1.3` | Set in postgresql.conf |
| Public schema CREATE revoked | `\dn+ public` (psql) | No `=C` for PUBLIC | `REVOKE CREATE ON SCHEMA public FROM PUBLIC` |
| Connections limited per role | `rolconnlimit > 0` on all app roles | > 0 | `ALTER ROLE app SET CONNECTION LIMIT 100` |
| Password expiry on app roles | `rolvaliduntil IS NOT NULL` | Non-null | `ALTER ROLE app VALID UNTIL '2026-12-31'` |
| Log connections enabled | `SHOW log_connections` | `on` | Set in postgresql.conf |

---

## Reference Files

- [Roles and privileges guide](references/roles-privileges.md) — role system, GRANT syntax, default privileges, predefined roles
- [pg_hba.conf guide](references/pg-hba-guide.md) — configuration format, methods, examples, reloading
- [Row-level security guide](references/rls-guide.md) — RLS policy syntax, patterns, and multi-tenant examples

## Examples

- [Security audit and hardening scenarios](examples/examples.md) — privilege audit, public schema fix, tenant RLS, SSL configuration
