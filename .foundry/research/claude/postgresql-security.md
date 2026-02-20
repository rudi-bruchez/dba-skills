# PostgreSQL Security Best Practices

## Overview
PostgreSQL security operates in layers: network access (pg_hba.conf), authentication, authorization (roles & privileges), object-level security (grants, row-level security), and transport encryption (SSL/TLS). Defense in depth requires all layers to be properly configured.

---

## Authentication — pg_hba.conf

### File Location & Format
```bash
# Default locations
/etc/postgresql/16/main/pg_hba.conf        # Debian/Ubuntu
/var/lib/pgsql/data/pg_hba.conf           # RHEL/CentOS
# Or: SHOW hba_file;
```

### pg_hba.conf Format
```
# TYPE  DATABASE  USER  ADDRESS       METHOD  OPTIONS
local   all       all                 scram-sha-256
host    all       all   127.0.0.1/32  scram-sha-256
host    all       all   ::1/128       scram-sha-256
```

Field meanings:
- **TYPE**: `local` (Unix socket), `host` (TCP/IP), `hostssl` (SSL required), `hostnossl`, `hostgssenc`
- **DATABASE**: database name, `all`, `sameuser`, `samerole`, `replication`, or comma-separated
- **USER**: username, `all`, `+groupname`, or comma-separated
- **ADDRESS**: IP/CIDR, hostname, or blank for local
- **METHOD**: `scram-sha-256`, `md5`, `password`, `trust`, `reject`, `peer`, `gss`, `ldap`, `cert`

### Recommended pg_hba.conf Configuration
```
# TYPE  DATABASE        USER            ADDRESS          METHOD
# Local connections via Unix socket
local   all             postgres                         peer
local   all             all                              scram-sha-256

# IPv4 loopback
host    all             all             127.0.0.1/32     scram-sha-256

# IPv6 loopback
host    all             all             ::1/128          scram-sha-256

# Replication connections (specific IP range)
host    replication     replicator      10.0.1.0/24      scram-sha-256

# Application connections (specific network)
hostssl all             appuser         10.0.0.0/8       scram-sha-256

# Reject everything else
host    all             all             0.0.0.0/0        reject
```

### Authentication Methods
| Method | Security | Notes |
|--------|---------|-------|
| `scram-sha-256` | Highest | Default since PG14; use for all new setups |
| `md5` | Moderate | Legacy; vulnerable to replay attacks; avoid |
| `password` | Low | Cleartext; never use over network |
| `trust` | None | Never use except for localhost postgres role |
| `peer` | High | Unix socket only; validates OS username |
| `cert` | Very High | Requires client SSL certificates |
| `ldap` | Varies | Delegates to LDAP server |
| `reject` | N/A | Always deny; useful for blacklisting |

### Reloading pg_hba.conf
```sql
-- Reload without restart
SELECT pg_reload_conf();
-- Or from OS:
-- pg_ctl reload
-- systemctl reload postgresql

-- View current pg_hba rules
SELECT * FROM pg_hba_file_rules;  -- PG10+
```

---

## Roles & Privileges

### Role Hierarchy Concepts
- Every user is a role with LOGIN privilege
- Roles can inherit from other roles (group roles)
- `SUPERUSER` bypasses all privilege checks (avoid!)
- `CREATEROLE`, `CREATEDB`, `REPLICATION` are specific capabilities

### Creating Roles
```sql
-- Application read-only role
CREATE ROLE app_readonly WITH
    LOGIN
    PASSWORD 'strong_random_password'
    CONNECTION LIMIT 20
    VALID UNTIL '2026-12-31';

-- Application read-write role
CREATE ROLE app_readwrite WITH LOGIN PASSWORD 'strong_password';

-- DBA role (no login, used as group)
CREATE ROLE dba_team;
GRANT dba_team TO alice, bob;

-- Replication role
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'rep_password';

-- Monitoring role (PG10+ predefined roles available)
GRANT pg_monitor TO monitoring_user;   -- Read-only access to monitoring views
```

### Predefined Roles (PostgreSQL 14+)
```sql
-- Grant monitoring access
GRANT pg_monitor TO grafana_user;

-- Read-only access to all tables (PG14+)
GRANT pg_read_all_data TO readonly_user;

-- Write access to all tables (PG14+)
GRANT pg_write_all_data TO etl_user;

-- Signal other processes
GRANT pg_signal_backend TO ops_user;

-- View statistics
GRANT pg_read_all_stats TO monitoring_user;
```

### Schema-Level Privileges
```sql
-- Create schemas for isolation
CREATE SCHEMA app AUTHORIZATION app_owner;
CREATE SCHEMA audit AUTHORIZATION dba;

-- Grant schema usage
GRANT USAGE ON SCHEMA app TO app_readonly, app_readwrite;

-- Default privileges (applied to future objects)
ALTER DEFAULT PRIVILEGES IN SCHEMA app
    GRANT SELECT ON TABLES TO app_readonly;

ALTER DEFAULT PRIVILEGES IN SCHEMA app
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_readwrite;

ALTER DEFAULT PRIVILEGES IN SCHEMA app
    GRANT USAGE ON SEQUENCES TO app_readwrite;

ALTER DEFAULT PRIVILEGES IN SCHEMA app
    GRANT EXECUTE ON FUNCTIONS TO app_readonly, app_readwrite;
```

### Table-Level Privileges
```sql
-- Grant specific table privileges
GRANT SELECT ON app.orders TO app_readonly;
GRANT SELECT, INSERT, UPDATE ON app.orders TO app_readwrite;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA app TO app_readwrite;

-- Column-level privileges
GRANT SELECT (id, name, email) ON users TO limited_user;
-- Deny access to sensitive column by granting only specific columns

-- Sequence privileges
GRANT USAGE, SELECT ON SEQUENCE orders_id_seq TO app_readwrite;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA app TO app_readwrite;

-- Revoke privileges
REVOKE DELETE ON app.orders FROM app_readwrite;

-- Revoke PUBLIC schema access (important security step)
REVOKE CREATE ON SCHEMA public FROM PUBLIC;
REVOKE ALL ON DATABASE mydb FROM PUBLIC;
```

### Viewing Privileges
```sql
-- Table privileges
SELECT grantee, privilege_type, table_name
FROM information_schema.role_table_grants
WHERE table_schema = 'app'
ORDER BY table_name, grantee, privilege_type;

-- Column privileges
SELECT grantee, table_name, column_name, privilege_type
FROM information_schema.role_column_grants
WHERE table_schema = 'app';

-- Schema privileges
SELECT nspname, nspacl
FROM pg_namespace
WHERE nspname NOT IN ('pg_catalog', 'information_schema');

-- Effective role memberships
SELECT rolname, member::regrole
FROM pg_auth_members
JOIN pg_roles ON pg_roles.oid = roleid;

-- Current user's roles
SELECT * FROM pg_roles WHERE pg_has_role(current_user, oid, 'member');
```

---

## Row Level Security (RLS)

### Enabling RLS
```sql
-- Enable RLS on a table (owner still bypasses by default)
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Force RLS even for table owner
ALTER TABLE orders FORCE ROW LEVEL SECURITY;
```

### Creating Policies
```sql
-- Policy: users can only see their own orders
CREATE POLICY orders_isolation ON orders
    USING (customer_id = current_setting('app.current_user_id')::int);

-- Separate SELECT and modification policies
CREATE POLICY orders_select ON orders
    FOR SELECT
    USING (customer_id = current_setting('app.current_user_id')::int);

CREATE POLICY orders_insert ON orders
    FOR INSERT
    WITH CHECK (customer_id = current_setting('app.current_user_id')::int);

CREATE POLICY orders_update ON orders
    FOR UPDATE
    USING (customer_id = current_setting('app.current_user_id')::int)
    WITH CHECK (customer_id = current_setting('app.current_user_id')::int);

-- Admin bypass policy
CREATE POLICY admin_all ON orders
    USING (pg_has_role(current_user, 'admin', 'member'));

-- Policy using a security-definer function
CREATE OR REPLACE FUNCTION current_tenant_id() RETURNS int
    LANGUAGE sql SECURITY DEFINER STABLE
    AS $$ SELECT current_setting('app.tenant_id', true)::int $$;

CREATE POLICY tenant_isolation ON orders
    USING (tenant_id = current_tenant_id());
```

### Setting Application Context
```sql
-- Set per-session variable (application sets this on connect)
SET app.current_user_id = '42';
SET app.tenant_id = '100';

-- Or use SET LOCAL within a transaction
BEGIN;
SET LOCAL app.current_user_id = '42';
-- All queries in this transaction see only user 42's rows
COMMIT;
```

### Viewing RLS Policies
```sql
SELECT tablename, policyname, permissive, roles, cmd, qual, with_check
FROM pg_policies
WHERE schemaname = 'public'
ORDER BY tablename, policyname;
```

---

## SSL/TLS Configuration

### Enabling SSL in postgresql.conf
```ini
ssl = on
ssl_cert_file = 'server.crt'           # Or full path
ssl_key_file = 'server.key'
ssl_ca_file = 'root.crt'               # CA cert for client cert auth
ssl_crl_file = ''                      # Certificate revocation list
ssl_min_protocol_version = 'TLSv1.2'  # Minimum TLS version (TLSv1.3 preferred)
ssl_ciphers = 'HIGH:MEDIUM:+3DES:!aNULL'  # Cipher suite
```

### Generating Self-Signed Certificates (Development Only)
```bash
# Generate CA key and cert
openssl genrsa -out ca.key 4096
openssl req -new -x509 -days 3650 -key ca.key -out ca.crt \
  -subj "/CN=PostgreSQL CA"

# Generate server key and cert
openssl genrsa -out server.key 2048
chmod 600 server.key
openssl req -new -key server.key -out server.csr \
  -subj "/CN=db.example.com"
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out server.crt -days 365

# Copy to PostgreSQL data directory
cp server.crt server.key /var/lib/postgresql/data/
chown postgres:postgres /var/lib/postgresql/data/server.*
```

### Client SSL Connection
```bash
# Require SSL in connection string
psql "host=db.example.com dbname=mydb user=app sslmode=require"

# Verify server certificate
psql "host=db.example.com dbname=mydb user=app sslmode=verify-full \
      sslrootcert=/path/to/ca.crt"

# Client certificate authentication
psql "host=db.example.com dbname=mydb user=app sslmode=verify-full \
      sslrootcert=ca.crt sslcert=client.crt sslkey=client.key"
```

### SSL Modes
| sslmode | Encryption | Certificate Verification |
|---------|-----------|------------------------|
| `disable` | No | No |
| `allow` | Tries SSL | No |
| `prefer` | Prefers SSL | No |
| `require` | Required | No |
| `verify-ca` | Required | Validates CA |
| `verify-full` | Required | Validates CA + hostname |

**Production minimum**: `verify-ca` or `verify-full`.

### Checking SSL Status
```sql
SELECT ssl, version, cipher, bits, client_dn, client_serial
FROM pg_stat_ssl
JOIN pg_stat_activity ON pg_stat_ssl.pid = pg_stat_activity.pid
WHERE pg_stat_activity.datname = current_database();
```

---

## Security Hardening Checklist

### Database Configuration
```sql
-- Review superuser accounts (should be minimal)
SELECT rolname, rolsuper, rolcreaterole, rolcreatedb, rolcanlogin
FROM pg_roles
WHERE rolsuper = true;

-- Review roles that can login
SELECT rolname, rolsuper, rolcreaterole, rolcreatedb, rolreplication,
       rolconnlimit, rolvaliduntil
FROM pg_roles
WHERE rolcanlogin = true
ORDER BY rolname;

-- Check for roles with no password expiry
SELECT rolname, rolvaliduntil
FROM pg_roles
WHERE rolcanlogin AND rolvaliduntil IS NULL;

-- Revoke dangerous PUBLIC defaults
REVOKE CREATE ON SCHEMA public FROM PUBLIC;
REVOKE ALL ON DATABASE mydb FROM PUBLIC;
-- Grant back minimally:
GRANT CONNECT ON DATABASE mydb TO app_role;

-- Ensure pg_catalog is not writable by non-superusers
-- (This is the default, just verify)
SHOW search_path;  -- Should not start with "$user" for shared DBs
```

### Important Security Parameters (postgresql.conf)
```ini
# Log all connections and disconnections
log_connections = on
log_disconnections = on

# Log all statements (careful — can be very verbose)
log_statement = 'ddl'    # Log all DDL; 'mod' for DML; 'all' for everything

# Log slow queries
log_min_duration_statement = 1000   # Log queries > 1 second

# Log lock waits
log_lock_waits = on
deadlock_timeout = 1000   # Log deadlocks after 1 second

# Prevent privilege escalation via COPY TO/FROM
# Allow superusers only by default (don't change)

# Restrict search_path to prevent schema injection
# Set in role definition:
-- ALTER ROLE myapp SET search_path = myschema, public;
```

### Privilege Audit Queries
```sql
-- Tables accessible by a specific role
SELECT table_schema, table_name, privilege_type
FROM information_schema.role_table_grants
WHERE grantee = 'app_user'
ORDER BY table_schema, table_name;

-- All superusers
SELECT usename FROM pg_user WHERE usesuper = true;

-- Roles with CREATEROLE (can escalate privileges)
SELECT rolname FROM pg_roles WHERE rolcreaterole = true;

-- Objects owned by each role
SELECT relowner::regrole AS owner,
       relname,
       relkind
FROM pg_class
WHERE relkind IN ('r', 'v', 'f')
  AND relnamespace NOT IN (
      SELECT oid FROM pg_namespace
      WHERE nspname IN ('pg_catalog', 'information_schema')
  )
ORDER BY owner, relname;
```

---

## Secrets & Password Management

### Password Policies
```sql
-- Set password with expiry
ALTER ROLE app_user PASSWORD 'new_strong_password' VALID UNTIL '2026-12-31';

-- Require password change
ALTER ROLE app_user PASSWORD 'temp_password' VALID UNTIL 'now';

-- Check pg_cryptopasswords extension for strength checks (PG17+)
-- Or use external tools
```

### Using .pgpass for Automation
```bash
# ~/.pgpass format: hostname:port:database:username:password
echo "db.example.com:5432:mydb:app_user:my_password" >> ~/.pgpass
chmod 600 ~/.pgpass
```

### Environment Variables (Avoid in Production Scripts)
```bash
# For scripts (prefer .pgpass or pg_service.conf)
export PGPASSWORD='password'   # Visible in process list
export PGSERVICE='mydb'        # Better: use service file

# ~/.pg_service.conf
# [mydb]
# host=db.example.com
# port=5432
# dbname=production
# user=app_user
```

---

## Audit Logging

### Using pgaudit Extension
```ini
# postgresql.conf
shared_preload_libraries = 'pgaudit'
pgaudit.log = 'write, ddl'    # Log write and DDL statements
pgaudit.log_catalog = off     # Don't log catalog queries
pgaudit.log_parameter = on    # Log bound parameters
pgaudit.log_relation = on     # Log relation name per statement
```

```sql
-- Enable pgaudit extension
CREATE EXTENSION pgaudit;

-- Session-level audit
SET pgaudit.log = 'all';

-- Role-level audit (audit specific role)
ALTER ROLE sensitive_user SET pgaudit.log = 'all';
```

### Log Analysis Queries
```sql
-- Review recent connection attempts from pg_log
-- (Parse log file or use log_destination = 'csvlog' for structured output)

-- Check current audit settings
SHOW pgaudit.log;
SHOW log_statement;
SHOW log_connections;
```

---

## Common Security Mistakes & Fixes

| Mistake | Risk | Fix |
|---------|------|-----|
| `trust` authentication | Anyone can connect as any user | Use `scram-sha-256` |
| Superuser for application | Total access if compromised | Create least-privilege role |
| Unused `md5` auth | Replay attacks | Migrate to `scram-sha-256` |
| `sslmode=disable` in app | Data in cleartext | Use `sslmode=require` minimum |
| PUBLIC schema CREATE | Schema injection attacks | `REVOKE CREATE ON SCHEMA public FROM PUBLIC` |
| Shared application user | No per-user audit trail | Create per-service users |
| Passwords in connection strings | Exposed in logs/env | Use .pgpass or pg_service.conf |
| No row-level security | All users see all data | Implement RLS for multi-tenant |
| No password expiry | Compromised passwords persist | Set `VALID UNTIL` on roles |
| Broad GRANT to PUBLIC | Unintended access | Audit and revoke PUBLIC grants |
