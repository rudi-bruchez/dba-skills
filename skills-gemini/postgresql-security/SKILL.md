---
name: postgresql-security
description: Specialized skill for securing PostgreSQL instances, managing authentication, authorization, and data encryption.
version: 1.0.0
tags:
  - postgresql
  - dba
  - security
  - encryption
  - auditing
---

# PostgreSQL Security

This skill provides expert guidance for securing PostgreSQL instances, protecting sensitive data, and ensuring compliance with security standards.

## Core Capabilities

- **Authentication Management:** Configuring `pg_hba.conf` and SCRAM-SHA-256 authentication.
- **Authorization & RBAC:** Implementing role-based access control and the principle of least privilege.
- **Row-Level Security (RLS):** Defining policies to restrict data access at the row level.
- **Data Encryption:** Configuring SSL/TLS for transit and pgcrypto for column-level encryption.
- **Auditing & Compliance:** Setting up `pgaudit` and monitoring for security events.

## Workflow: Security Hardening

1.  **Enforce Strong Authentication:**
    - Use `scram-sha-256` for all password authentication.
    - Configure `pg_hba.conf` to restrict access to trusted hosts and databases.
2.  **Implement Least Privilege:**
    - Use roles and groups for granting permissions.
    - Revoke unnecessary privileges from the `public` role.
    - Avoid using the `postgres` superuser for routine tasks.
3.  **Secure Data in Transit & at Rest:**
    - Enable SSL/TLS for all client-to-server connections and force encryption on the server.
    - Use file-system-level encryption (e.g., LUKS, EBS) for data at rest.
4.  **Configure Auditing:**
    - Install and configure the `pgaudit` extension to track DDL and DML operations.
5.  **Enable Row-Level Security:**
    - Use RLS to restrict which rows a user can see based on their identity or role.

## Essential Commands

### 1. Create a Role and Grant Permissions
```sql
CREATE ROLE reporting_user WITH LOGIN PASSWORD 'StrongPassword123!';
GRANT SELECT ON ALL TABLES IN SCHEMA public TO reporting_user;
```

### 2. Enable Row-Level Security (RLS)
```sql
-- 1. Enable RLS on a table
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
-- 2. Create a policy for users to see only their own rows
CREATE POLICY user_orders_policy ON orders
FOR SELECT
TO public
USING (user_id = current_user);
```

### 3. Check for Superuser Privileges
```sql
SELECT usename, usecreatedb, usesuper, usecatupd, userepl
FROM pg_user
WHERE usesuper = true;
```

## Best Practices (2024-2025)

- **SCRAM-SHA-256 by Default:** Ensure all new users use this authentication method.
- **SSL/TLS for All Connections:** Enforce encryption between applications and the database.
- **Principle of Least Privilege:** Avoid using the `postgres` superuser for routine tasks.
- **Surface Area Reduction:** Disable unused extensions and set `listen_addresses` to the minimum necessary network interfaces.
- **SQL Injection Prevention:** Use parameterized queries in application code.

## References

- [pg-hba Guide](./references/pg-hba-guide.md)
- [RLS Guide](./references/rls-guide.md)
- [Roles & Privileges](./references/roles-privileges.md)
---
