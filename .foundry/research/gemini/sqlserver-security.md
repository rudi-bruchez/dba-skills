# SQL Server Security Research

## Overview
SQL Server security focuses on protecting data from unauthorized access while ensuring it's available for legitimate use. This involves a defense-in-depth approach covering authentication, authorization, encryption, and auditing.

## Authentication
- **Windows Authentication:** The most secure method, leveraging Active Directory or local Windows accounts. Recommended by Microsoft.
- **SQL Server Authentication:** Uses usernames and passwords stored within the database instance. Not as secure, but sometimes necessary for legacy apps or cloud-to-local connections.
- **Entra ID (formerly Azure AD) Integration:** Modern authentication for SQL Server, supporting MFA (Multi-Factor Authentication) and modern identity management.

## Authorization (Least Privilege)
- **Logins vs. Users:** Logins are for instance-level access; Users are for database-level access.
- **Fixed Server Roles:** `sysadmin`, `serveradmin`, `securityadmin`, etc. Grant these sparingly.
- **Fixed Database Roles:** `db_owner`, `db_datareader`, `db_datawriter`, `db_denydatawriter`, etc.
- **Granular Permissions:** Grant only the specific permissions a user or role needs (e.g., `GRANT SELECT ON Schema.Table TO Role`).

## Data Encryption
- **Data in Transit:** Use SSL/TLS for all connections. Force encryption on the server side (`ForceEncryption=Yes`).
- **Data at Rest (TDE):** Transparent Data Encryption encrypts the database files (`.mdf`, `.ndf`, `.ldf`) and backups. Operates at the page level.
- **Always Encrypted:** End-to-end encryption. Data is encrypted/decrypted at the client level, meaning SQL Server never sees the plaintext.
- **Dynamic Data Masking (DDM):** Obfuscates data for non-privileged users without changing the underlying data (e.g., masking credit card numbers).

## Auditing & Compliance
- **SQL Server Audit:** Tracks server-level and database-level events (e.g., failed logins, schema changes, data access).
- **Extended Events:** Low-overhead tracing that can be used for security auditing.
- **Compliance Standards:** Map SQL Server security controls to standards like GDPR, HIPAA, SOC 2, or PCI DSS.

## Best Practices (2024-2025)
- **MFA (Multi-Factor Authentication):** Enforce MFA for all administrative accounts (especially via Entra ID).
- **Surface Area Reduction:** Disable unused features like `xp_cmdshell`, OLE Automation, and the SQL Browser service if not needed.
- **Row-Level Security (RLS):** Implement RLS to restrict which rows a user can see based on their identity or role.
- **SQL Injection Prevention:** Use parameterized queries or stored procedures instead of dynamic SQL.
- **Regular Security Audits:** Use tools like `sp_BlitzSecurity` or specialized scanners (e.g., SQL vulnerability assessment in Azure).

## Key T-SQL Commands
- `CREATE LOGIN [LoginName] WITH PASSWORD = 'StrongPassword';`
- `CREATE USER [UserName] FOR LOGIN [LoginName];`
- `GRANT SELECT ON [TableName] TO [UserName];`
- `EXEC sp_configure 'show advanced options', 1; RECONFIGURE; EXEC sp_configure 'xp_cmdshell', 0; RECONFIGURE;`

## References
- [Microsoft Documentation: SQL Server Security](https://learn.microsoft.com/en-us/sql/relational-databases/security/sql-server-security)
- [SQL Server Security Best Practices (Checklist)](https://www.netwrix.com/sql_server_security_best_practices.html)
- [Brent Ozar's SQL Server Security Guide](https://www.brentozar.com/archive/2016/11/sql-server-security-best-practices/)
