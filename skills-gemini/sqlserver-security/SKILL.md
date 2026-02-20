---
name: sqlserver-security
description: Specialized skill for securing SQL Server instances, managing authentication, authorization, and data encryption.
version: 1.0.0
tags:
  - sqlserver
  - dba
  - security
  - encryption
  - auditing
---

# SQL Server Security

This skill provides expert guidance for securing SQL Server instances, protecting sensitive data, and ensuring compliance with security standards.

## Core Capabilities

- **Authentication Management:** Configuring Windows, SQL Server, and Entra ID authentication.
- **Authorization & Permissions:** Implementing the principle of least privilege using roles and granular permissions.
- **Data Encryption:** Configuring TDE, Always Encrypted, and Dynamic Data Masking.
- **Auditing & Compliance:** Setting up SQL Server Audit and monitoring for security events.
- **Surface Area Reduction:** Disabling unused features and securing the server environment.

## Workflow: Security Hardening

1.  **Enforce Strong Authentication:**
    - Prefer Windows Authentication or Entra ID for all users and services.
    - If SQL Server Authentication is necessary, enforce strong password policies.
2.  **Implement Least Privilege:**
    - Use database roles (`db_datareader`, `db_datawriter`) for most users.
    - Grant granular permissions only where necessary (e.g., `GRANT SELECT ON Schema.Table TO Role`).
    - Avoid using `sysadmin` for routine tasks.
3.  **Secure Data at Rest & in Transit:**
    - Enable Transparent Data Encryption (TDE) for sensitive databases.
    - Use SSL/TLS for all client-to-server connections and force encryption on the server.
4.  **Configure Auditing:**
    - Set up SQL Server Audit to track failed logins, schema changes, and sensitive data access.
5.  **Reduce Surface Area:**
    - Disable features like `xp_cmdshell` and OLE Automation if not needed.
    - Ensure the SQL Server service accounts have minimal permissions on the host OS.

## Essential Commands

### 1. Create a Login and User
```sql
CREATE LOGIN [LoginName] WITH PASSWORD = 'StrongPassword123!', CHECK_POLICY = ON;
CREATE USER [UserName] FOR LOGIN [LoginName];
GRANT SELECT ON [TableName] TO [UserName];
```

### 2. Enable Transparent Data Encryption (TDE)
```sql
-- 1. Create a Master Key
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'MasterKeyPassword123!';
-- 2. Create a Certificate
CREATE CERTIFICATE TDE_Cert WITH SUBJECT = 'TDE Encryption Certificate';
-- 3. Create a Database Encryption Key (DEK)
USE [MyDatabase];
CREATE DATABASE ENCRYPTION KEY WITH ALGORITHM = AES_256 ENCRYPTION BY SERVER CERTIFICATE TDE_Cert;
-- 4. Enable Encryption
ALTER DATABASE [MyDatabase] SET ENCRYPTION ON;
```

### 3. Check for Elevated Permissions
```sql
SELECT
    p.name AS PrincipalName,
    p.type_desc AS PrincipalType,
    r.name AS RoleName
FROM sys.server_principals AS p
JOIN sys.server_role_members AS rm ON p.principal_id = rm.member_principal_id
JOIN sys.server_principals AS r ON rm.role_principal_id = r.principal_id
WHERE r.name = 'sysadmin';
```

## Best Practices (2024-2025)

- **MFA (Multi-Factor Authentication):** Enforce MFA for all administrative accounts (especially via Entra ID).
- **Always Encrypted:** Use Always Encrypted for highly sensitive columns (e.g., SSN, Credit Card numbers).
- **SQL Vulnerability Assessment:** Regularly run the SQL vulnerability assessment tool in Azure or SSMS.
- **Row-Level Security (RLS):** Implement RLS to restrict row access based on user identity or role.
- **Dynamic Data Masking (DDM):** Obfuscate sensitive data for non-privileged users.

## References

- [Audit Configuration](./references/audit-configuration.md)
- [Encryption Guide](./references/encryption-guide.md)
- [Permissions Model](./references/permissions-model.md)
