# PostgreSQL Roles and Privileges Reference

## Role System Concepts

- Every user is a **role** with `LOGIN` privilege
- Roles can belong to other roles (group membership)
- Privileges flow down through group membership
- `SUPERUSER` bypasses all access checks — avoid for application accounts
- Principle of least privilege: grant only what is needed

## Creating Roles

```sql
-- Application read-only role
CREATE ROLE app_readonly WITH
    LOGIN
    PASSWORD 'strong_random_password_here'
    CONNECTION LIMIT 50
    VALID UNTIL '2026-12-31';

-- Application read-write role
CREATE ROLE app_readwrite WITH
    LOGIN
    PASSWORD 'strong_password'
    CONNECTION LIMIT 100;

-- Group role (no login, used for privilege grouping)
CREATE ROLE developers;
GRANT developers TO alice, bob;

-- Replication role
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'rep_password';

-- Monitoring role (uses predefined pg_monitor)
CREATE ROLE monitoring_user WITH LOGIN PASSWORD 'monitor_pass';
GRANT pg_monitor TO monitoring_user;

-- Service account with fixed expiry
CREATE ROLE etl_service WITH
    LOGIN
    PASSWORD 'service_password'
    CONNECTION LIMIT 10
    VALID UNTIL '2026-06-30';
```

## Altering Roles

```sql
-- Change password
ALTER ROLE app_user PASSWORD 'new_strong_password';

-- Change password with expiry
ALTER ROLE app_user PASSWORD 'new_password' VALID UNTIL '2026-12-31';

-- Remove superuser
ALTER ROLE username NOSUPERUSER;

-- Set connection limit
ALTER ROLE app_user CONNECTION LIMIT 50;

-- Set role-level configuration
ALTER ROLE reporter SET work_mem = '512MB';
ALTER ROLE reporter SET random_page_cost = 1.1;
ALTER ROLE app_user SET search_path = myapp, public;
ALTER ROLE app_user SET idle_in_transaction_session_timeout = '300000';
ALTER ROLE app_user SET statement_timeout = '30000';  -- 30s max query time

-- Lock a role (disable login)
ALTER ROLE username NOLOGIN;
```

## Dropping Roles

```sql
-- Must reassign owned objects first
REASSIGN OWNED BY old_user TO postgres;
DROP OWNED BY old_user;     -- Remove all privileges granted to old_user
DROP ROLE old_user;
```

---

## Predefined Roles (PostgreSQL 14+)

| Role | What it grants |
|------|---------------|
| `pg_read_all_data` | SELECT on all tables, views, sequences in all schemas |
| `pg_write_all_data` | INSERT, UPDATE, DELETE on all tables in all schemas |
| `pg_read_all_settings` | Read `pg_settings`, `pg_file_settings`, `pg_hba_file_rules` |
| `pg_read_all_stats` | Read `pg_stat_*` views without restrictions |
| `pg_stat_scan_tables` | Execute `pg_stat_get_*` functions |
| `pg_monitor` | All monitoring privileges (combines read_all_settings + read_all_stats + stat_scan_tables) |
| `pg_signal_backend` | `pg_cancel_backend()` and `pg_terminate_backend()` for non-superuser backends |
| `pg_read_server_files` | COPY FROM server files; access server file functions |
| `pg_write_server_files` | COPY TO server files |
| `pg_execute_server_program` | COPY with PROGRAM |
| `pg_checkpoint` | `pg_start_backup()`, `pg_stop_backup()`, `pg_switch_wal()` |

```sql
-- Grant monitoring access to a monitoring agent
GRANT pg_monitor TO grafana_user;

-- Grant read-only access across all tables
GRANT pg_read_all_data TO readonly_user;

-- Grant ability to signal backends
GRANT pg_signal_backend TO ops_user;
```

---

## Schema-Level Privileges

### Setup Pattern (New Database)

```sql
-- 1. Create schema with explicit owner
CREATE SCHEMA myapp AUTHORIZATION app_owner;

-- 2. Revoke dangerous defaults from PUBLIC
REVOKE CREATE ON SCHEMA public FROM PUBLIC;
REVOKE ALL ON DATABASE mydb FROM PUBLIC;

-- 3. Grant schema usage
GRANT USAGE ON SCHEMA myapp TO app_readonly, app_readwrite;

-- 4. Set default privileges for future objects
-- (Set by the role that will own the objects, usually app_owner)
SET ROLE app_owner;
ALTER DEFAULT PRIVILEGES IN SCHEMA myapp
    GRANT SELECT ON TABLES TO app_readonly;
ALTER DEFAULT PRIVILEGES IN SCHEMA myapp
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_readwrite;
ALTER DEFAULT PRIVILEGES IN SCHEMA myapp
    GRANT USAGE ON SEQUENCES TO app_readwrite;
ALTER DEFAULT PRIVILEGES IN SCHEMA myapp
    GRANT EXECUTE ON FUNCTIONS TO app_readonly, app_readwrite;
RESET ROLE;
```

### Grant Existing Objects

```sql
-- All existing tables in a schema
GRANT SELECT ON ALL TABLES IN SCHEMA myapp TO app_readonly;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA myapp TO app_readwrite;

-- All sequences in a schema
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA myapp TO app_readwrite;

-- All functions in a schema
GRANT EXECUTE ON ALL FUNCTIONS IN SCHEMA myapp TO app_readonly;

-- Specific table
GRANT SELECT ON myapp.orders TO app_readonly;
GRANT SELECT, INSERT, UPDATE ON myapp.orders TO app_readwrite;
GRANT SELECT, INSERT, UPDATE, DELETE ON myapp.orders TO app_readwrite;

-- Revoke a specific privilege
REVOKE DELETE ON myapp.orders FROM app_readwrite;
```

### Column-Level Privileges

```sql
-- Grant access to specific columns only (hides sensitive columns like SSN, salary)
GRANT SELECT (id, name, email, created_at) ON users TO limited_user;
-- limited_user cannot see the 'salary' or 'ssn' columns

-- Check column privileges
SELECT grantee, table_name, column_name, privilege_type
FROM information_schema.role_column_grants
WHERE table_schema = 'myapp'
ORDER BY table_name, column_name;
```

---

## Viewing Privileges

```sql
-- Table privileges
SELECT grantee, privilege_type, table_schema, table_name
FROM information_schema.role_table_grants
WHERE table_schema = 'myapp'
ORDER BY table_schema, table_name, grantee;

-- Schema privileges
SELECT nspname AS schema,
       nspacl AS access_control_list
FROM pg_namespace
WHERE nspname NOT IN ('pg_catalog', 'information_schema', 'pg_toast');

-- Role memberships
SELECT r.rolname AS role, m.rolname AS member
FROM pg_auth_members am
JOIN pg_roles r ON r.oid = am.roleid
JOIN pg_roles m ON m.oid = am.member
ORDER BY r.rolname, m.rolname;

-- All privileges on all objects for a role
SELECT table_schema, table_name, privilege_type
FROM information_schema.role_table_grants
WHERE grantee = 'app_user'
ORDER BY table_schema, table_name;

-- Objects owned by a role
SELECT relowner::regrole AS owner, relname, relkind
FROM pg_class
WHERE relkind IN ('r', 'v', 'm', 'f')
  AND relnamespace NOT IN (
      SELECT oid FROM pg_namespace WHERE nspname IN ('pg_catalog', 'information_schema')
  )
ORDER BY owner, relname;

-- Check specific privilege
SELECT has_table_privilege('app_user', 'myapp.orders', 'SELECT');
SELECT has_schema_privilege('app_user', 'myapp', 'USAGE');
SELECT has_column_privilege('app_user', 'users', 'salary', 'SELECT');
```

---

## Default Privileges Reference

`ALTER DEFAULT PRIVILEGES` configures privileges automatically applied to future objects.

```sql
-- Show current default privileges
SELECT pg_get_userbyid(defaclrole) AS role,
       nspname AS schema,
       defaclobjtype AS object_type,
       defaclacl AS acl
FROM pg_default_acl
LEFT JOIN pg_namespace ON pg_namespace.oid = defaclnamespace
ORDER BY role, schema;

-- Reset default privileges for a specific grantee
ALTER DEFAULT PRIVILEGES IN SCHEMA myapp
    REVOKE ALL ON TABLES FROM old_readonly_role;
```

---

## Password Management

```sql
-- Set password with expiry
ALTER ROLE app_user PASSWORD 'NewStr0ng!Pass' VALID UNTIL '2026-12-31';

-- Force immediate expiry (user must change on next login)
ALTER ROLE app_user VALID UNTIL 'now';

-- Check roles with no password expiry
SELECT rolname, rolvaliduntil
FROM pg_roles
WHERE rolcanlogin = true
  AND rolvaliduntil IS NULL
ORDER BY rolname;
```

### Using .pgpass for Automated Scripts

```bash
# ~/.pgpass format: hostname:port:database:username:password
echo "db.example.com:5432:mydb:app_user:strongpassword" >> ~/.pgpass
chmod 600 ~/.pgpass
# pg_dump will automatically use this for authentication
```

### pg_service.conf for Application Configuration

```ini
# ~/.pg_service.conf or /etc/pg_service.conf
[myapp-prod]
host=db.example.com
port=5432
dbname=myapp
user=app_user
sslmode=verify-full
sslrootcert=/etc/ssl/certs/db-ca.crt

[myapp-readonly]
host=db-replica.example.com
port=5432
dbname=myapp
user=app_readonly
sslmode=verify-full
```

```bash
# Connect using service name
psql service=myapp-prod
# Connection string:
# postgresql://service=myapp-prod
```
