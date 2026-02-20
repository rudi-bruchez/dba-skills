# PostgreSQL Security Research

## Overview
PostgreSQL security involves a layered defense, including network-level controls, client authentication, role-based access control, row-level security, and data encryption.

## Authentication
- **`pg_hba.conf`:** Configuration file that controls client authentication based on database, user, host, and method.
- **Authentication Methods:**
    - **`trust`:** No password required. Avoid in production.
    - **`scram-sha-256`:** Modern, cryptographically secure password authentication. Recommended for all new installations.
    - **`md5`:** Legacy password authentication. Use `scram-sha-256` instead.
    - **`peer` / `ident`:** Uses the OS user for authentication. Recommended for local connections.
    - **`cert`:** Client-certificate-based authentication for SSL connections.
    - **`ldap` / `pam` / `gssapi`:** For enterprise directory integration (e.g., AD, Kerberos).

## Authorization (RBAC)
- **Roles & Users:** Roles are users and/or groups. Roles can own objects and be granted permissions.
- **Granting Privileges:** `GRANT SELECT, INSERT ON schema.table TO role;`
- **Revoking Privileges:** `REVOKE ALL ON schema.table FROM role;`
- **`public` Schema:** Be aware of the default privileges granted to the `public` role and restrict access if necessary.
- **Default Privileges:** Use `ALTER DEFAULT PRIVILEGES` to ensure new objects are created with the correct permissions.

## Row-Level Security (RLS)
- **Definition:** Fine-grained access control that restricts row access based on a policy (e.g., `WHERE user_id = current_user`).
- **Required Steps:** `ALTER TABLE [table] ENABLE ROW LEVEL SECURITY;` and `CREATE POLICY [policy_name] ON [table] FOR ALL TO [role] USING (user_id = current_user);`

## Data Encryption
- **Data in Transit:** Use SSL/TLS for all client-to-server connections. Set `ssl = on` in `postgresql.conf`.
- **Data at Rest:**
    - **Transparent Data Encryption (TDE):** Not natively supported in vanilla PostgreSQL. Requires third-party tools or file-system-level encryption (e.g., LUKS, EBS encryption).
    - **Column-Level Encryption:** Use the `pgcrypto` extension for sensitive data within the database.

## Auditing & Monitoring
- **`pgaudit`:** Extension that provides detailed session and object-level logging for compliance and security auditing.
- **Logging Settings:** Use `log_connections`, `log_disconnections`, `log_statement = 'ddl'`, and `log_duration = on` to track activity.

## Best Practices (2024-2025)
- **SCRAM-SHA-256 by Default:** Ensure all new users use this authentication method.
- **SSL/TLS for All Connections:** Enforce encryption between applications and the database.
- **Principle of Least Privilege:** Avoid using the `postgres` superuser for routine tasks.
- **Surface Area Reduction:** Disable unused extensions and set `listen_addresses` to the minimum necessary network interfaces.
- **SQL Injection Prevention:** Use parameterized queries in application code.

## Key SQL Commands
- `CREATE ROLE [role_name] WITH LOGIN PASSWORD 'StrongPassword' CONNECTION LIMIT 10;`
- `GRANT [role_name] TO [user_name];`
- `ALTER TABLE [table_name] ENABLE ROW LEVEL SECURITY;`

## References
- [PostgreSQL Documentation: Client Authentication](https://www.postgresql.org/docs/current/auth-pg-hba-conf.html)
- [pgaudit: Documentation](https://www.pgaudit.org/)
- [PostgreSQL Security Best Practices (Checklist)](https://www.postgresql.org/docs/current/security.html)
