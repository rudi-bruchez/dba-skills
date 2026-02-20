# SQL Server Security Best Practices

## Overview

SQL Server security operates in layers: network, operating system, SQL Server instance, database, schema, and object level. The principle of least privilege governs all access decisions. Security encompasses authentication, authorization, encryption, auditing, and compliance.

---

## Authentication Modes

### Windows Authentication (Recommended)
- Uses Windows/Active Directory accounts and groups.
- Kerberos/NTLM authentication; no password stored in SQL Server.
- Supports group policy, password expiration, account lockout.
- Single sign-on (SSO) in domain environments.

### SQL Server Authentication
- SQL Server stores and manages login credentials.
- Password policies can be enforced (CHECK_POLICY, CHECK_EXPIRATION).
- Required for: non-domain clients, cross-domain scenarios, applications that cannot use Windows auth.
- Risk: credentials in connection strings; use secrets management tools.

### Mixed Mode
- Both Windows and SQL Authentication enabled.
- Required when SQL Server Authentication logins must be used.

```sql
-- Check current authentication mode
SELECT SERVERPROPERTY('IsIntegratedSecurityOnly') AS windows_auth_only;
-- 1 = Windows only, 0 = Mixed mode

-- Check SA account status (should be disabled if not needed)
SELECT name, is_disabled, is_policy_checked, is_expiration_checked
FROM sys.sql_logins WHERE name = 'sa';

-- Disable SA account
ALTER LOGIN sa DISABLE;

-- Rename SA (security through obscurity, but still recommended)
ALTER LOGIN sa WITH NAME = [sql_admin_renamed];
```

---

## Logins, Users, and Roles

### Principal Hierarchy
```
Instance Level:   Logins (Windows Login, SQL Login, Windows Group)
                      ↓
Database Level:   Database Users (mapped from logins)
                      ↓
Schema Level:     Schema ownership
                      ↓
Object Level:     Table, view, procedure permissions
```

### Logins (Instance Level)
```sql
-- Create a SQL Server login
CREATE LOGIN [AppUser] WITH PASSWORD = 'StrongPass123!',
    CHECK_POLICY = ON, CHECK_EXPIRATION = ON;

-- Create a Windows login
CREATE LOGIN [DOMAIN\AppServiceAccount] FROM WINDOWS;

-- Create a Windows group login
CREATE LOGIN [DOMAIN\DBATeam] FROM WINDOWS;

-- Disable a login
ALTER LOGIN [AppUser] DISABLE;

-- List all logins
SELECT
    name,
    type_desc,
    is_disabled,
    is_policy_checked,
    is_expiration_checked,
    create_date,
    modify_date,
    default_database_name
FROM sys.server_principals
WHERE type IN ('S', 'U', 'G')  -- SQL login, Windows login, Windows group
ORDER BY type_desc, name;
```

### Database Users
```sql
-- Create a user from a login
USE [YourDatabase];
CREATE USER [AppUser] FOR LOGIN [AppUser];

-- Create a user with default schema
CREATE USER [AppUser] FOR LOGIN [AppUser] WITH DEFAULT_SCHEMA = dbo;

-- Create a contained database user (no login required; SQL 2012+)
CREATE USER [ContainedUser] WITH PASSWORD = 'StrongPass123!';

-- Orphaned user check and fix
SELECT dp.name AS user_name, sp.name AS login_name
FROM sys.database_principals dp
LEFT JOIN sys.server_principals sp ON dp.sid = sp.sid
WHERE dp.type IN ('S', 'U')
  AND dp.name NOT IN ('dbo', 'guest', 'INFORMATION_SCHEMA', 'sys')
  AND sp.name IS NULL;  -- These are orphaned users

-- Fix orphaned user
ALTER USER [OrphanedUser] WITH LOGIN = [MatchingLogin];
```

### Fixed Server Roles
```sql
-- Key server roles (assign sparingly)
-- sysadmin:       Full control of SQL Server instance (use minimally)
-- serveradmin:    Server configuration, SHUTDOWN
-- securityadmin:  Manage logins and permissions (can escalate to sysadmin!)
-- processadmin:   Kill connections
-- dbcreator:      Create, alter, drop databases
-- bulkadmin:      Execute BULK INSERT
-- diskadmin:      Manage disk files
-- public:         Default role, all logins are members

-- Add login to server role
ALTER SERVER ROLE [sysadmin] ADD MEMBER [DOMAIN\DBAAccount];
ALTER SERVER ROLE [dbcreator] ADD MEMBER [DOMAIN\DevTeam];

-- List server role members
SELECT r.name AS role_name, m.name AS member_name, m.type_desc
FROM sys.server_role_members srm
JOIN sys.server_principals r ON srm.role_principal_id = r.principal_id
JOIN sys.server_principals m ON srm.member_principal_id = m.principal_id
ORDER BY r.name, m.name;
```

### Fixed Database Roles
```sql
-- Key database roles
-- db_owner:              Full control of database (use sparingly)
-- db_datareader:         SELECT on all tables
-- db_datawriter:         INSERT, UPDATE, DELETE on all tables
-- db_ddladmin:           CREATE, ALTER, DROP objects (no permissions management)
-- db_securityadmin:      Manage database roles and permissions
-- db_backupoperator:     Backup database
-- db_denydatareader:     Deny SELECT on all tables
-- db_denydatawriter:     Deny DML on all tables

-- Add user to database role
ALTER ROLE [db_datareader] ADD MEMBER [AppUser];
ALTER ROLE [db_datawriter] ADD MEMBER [AppUser];

-- Create custom database role (least-privilege pattern)
CREATE ROLE [AppReadRole];
GRANT SELECT ON SCHEMA::dbo TO [AppReadRole];
GRANT EXECUTE ON SCHEMA::dbo TO [AppReadRole];

-- Add user to custom role
ALTER ROLE [AppReadRole] ADD MEMBER [AppUser];
```

---

## Permissions (GRANT / DENY / REVOKE)

```sql
-- Object-level permissions
GRANT SELECT ON dbo.Customers TO [AppUser];
GRANT INSERT, UPDATE ON dbo.Orders TO [AppUser];
GRANT EXECUTE ON dbo.usp_GetOrderStatus TO [AppUser];
DENY DELETE ON dbo.Customers TO [AppUser];
REVOKE SELECT ON dbo.Customers FROM [AppUser];

-- Schema-level permissions (covers all objects in schema)
GRANT SELECT ON SCHEMA::Reporting TO [ReportUser];
GRANT EXECUTE ON SCHEMA::dbo TO [AppUser];

-- Database-level permissions
GRANT CREATE TABLE TO [DevUser];
GRANT VIEW DATABASE STATE TO [MonitorUser];
GRANT VIEW DEFINITION TO [DevUser];

-- Instance-level permissions
GRANT VIEW SERVER STATE TO [MonitorLogin];
GRANT ALTER TRACE TO [DBALogin];

-- WITH GRANT OPTION (allow grantee to grant to others - use carefully)
GRANT SELECT ON dbo.Products TO [AppUser] WITH GRANT OPTION;
```

### Permission Reporting
```sql
-- Effective permissions for a user in current database
EXECUTE AS USER = 'AppUser';
SELECT * FROM fn_my_permissions(NULL, 'DATABASE');
REVERT;

-- All permissions in database
SELECT
    dp.name AS principal_name,
    dp.type_desc AS principal_type,
    o.name AS object_name,
    o.type_desc AS object_type,
    p.permission_name,
    p.state_desc AS permission_state
FROM sys.database_permissions p
LEFT JOIN sys.database_principals dp ON p.grantee_principal_id = dp.principal_id
LEFT JOIN sys.objects o ON p.major_id = o.object_id
ORDER BY dp.name, o.name, p.permission_name;

-- Server-level permissions
SELECT
    sp.name AS login_name,
    sp.type_desc,
    srvp.permission_name,
    srvp.state_desc
FROM sys.server_permissions srvp
JOIN sys.server_principals sp ON srvp.grantee_principal_id = sp.principal_id
ORDER BY sp.name, srvp.permission_name;
```

---

## Encryption

### Transparent Data Encryption (TDE)
Encrypts data at rest: data files, log files, and backups.
```sql
-- Step 1: Create master key in master database
USE master;
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'StrongMasterKeyPass!';

-- Step 2: Create TDE certificate
CREATE CERTIFICATE TDECert WITH SUBJECT = 'TDE Certificate';

-- Step 3: Back up the certificate (MANDATORY before enabling TDE)
BACKUP CERTIFICATE TDECert
TO FILE = 'D:\Certs\TDECert.cer'
WITH PRIVATE KEY (
    FILE = 'D:\Certs\TDECert.pvk',
    ENCRYPTION BY PASSWORD = 'CertBackupPass!'
);

-- Step 4: Create database encryption key
USE [YourDatabase];
CREATE DATABASE ENCRYPTION KEY
WITH ALGORITHM = AES_256
ENCRYPTION BY SERVER CERTIFICATE TDECert;

-- Step 5: Enable TDE
ALTER DATABASE [YourDatabase] SET ENCRYPTION ON;

-- Monitor TDE encryption progress
SELECT db_name(database_id) AS database_name,
       encryption_state,
       encryption_state_desc,
       percent_complete
FROM sys.dm_database_encryption_keys;
-- encryption_state: 0=None, 1=Unencrypted, 2=Encryption in progress, 3=Encrypted, 4=Key change, 5=Decryption in progress
```

### Always Encrypted (Column-Level, client-side)
```sql
-- Encrypt specific columns; keys never leave client
-- Column master key stored in Windows Certificate Store, Azure Key Vault, etc.

-- Create column master key metadata
CREATE COLUMN MASTER KEY [CMK_AE]
WITH (
    KEY_STORE_PROVIDER_NAME = N'MSSQL_CERTIFICATE_STORE',
    KEY_PATH = N'CurrentUser/My/THUMBPRINT_HERE'
);

-- Create column encryption key
CREATE COLUMN ENCRYPTION KEY [CEK_AE]
WITH VALUES (
    COLUMN_MASTER_KEY = [CMK_AE],
    ALGORITHM = 'RSA_OAEP',
    ENCRYPTED_VALUE = 0x...  -- Generated by SSMS/PowerShell wizard
);

-- Encrypt a column (requires table rebuild)
ALTER TABLE dbo.Customers
ALTER COLUMN CreditCardNumber NVARCHAR(20)
    ENCRYPTED WITH (ENCRYPTION_TYPE = DETERMINISTIC,
                    ALGORITHM = 'AEAD_AES_256_CBC_HMAC_SHA_256',
                    COLUMN_ENCRYPTION_KEY = CEK_AE) NOT NULL;
```

### Cell-Level Encryption (symmetric key)
```sql
USE [YourDatabase];
-- Create symmetric key
CREATE SYMMETRIC KEY DataKey_AES256
WITH ALGORITHM = AES_256
ENCRYPTION BY PASSWORD = 'SymKeyPassword!';

-- Encrypt data
OPEN SYMMETRIC KEY DataKey_AES256 DECRYPTION BY PASSWORD = 'SymKeyPassword!';
UPDATE dbo.SensitiveData
SET EncryptedCol = EncryptByKey(Key_GUID('DataKey_AES256'), PlainTextCol);
CLOSE SYMMETRIC KEY DataKey_AES256;

-- Decrypt data
OPEN SYMMETRIC KEY DataKey_AES256 DECRYPTION BY PASSWORD = 'SymKeyPassword!';
SELECT CONVERT(NVARCHAR(200), DecryptByKey(EncryptedCol)) AS decrypted_value
FROM dbo.SensitiveData;
CLOSE SYMMETRIC KEY DataKey_AES256;
```

### Connections Encryption (TLS)
- Configure TLS certificate on SQL Server for encrypted connections.
- Force encryption at instance level: SQL Server Configuration Manager → SQL Server Network Configuration → Protocols → Force Encryption = Yes.
- Verify: `sys.dm_exec_connections` column `encrypt_option`.

```sql
SELECT session_id, encrypt_option, net_transport, client_net_address
FROM sys.dm_exec_connections;
```

---

## SQL Server Audit

### Server Audit (SQL 2008+)
```sql
-- Step 1: Create server audit (writes to file or Windows Event Log)
CREATE SERVER AUDIT [DBA_SecurityAudit]
TO FILE (
    FILEPATH = N'D:\Audit\',
    MAXSIZE = 100 MB,
    MAX_ROLLOVER_FILES = 10,
    RESERVE_DISK_SPACE = OFF
)
WITH (QUEUE_DELAY = 1000, ON_FAILURE = CONTINUE);

ALTER SERVER AUDIT [DBA_SecurityAudit] WITH (STATE = ON);

-- Step 2: Create audit specification (what to audit)
CREATE SERVER AUDIT SPECIFICATION [DBA_ServerAuditSpec]
FOR SERVER AUDIT [DBA_SecurityAudit]
ADD (FAILED_LOGIN_GROUP),
ADD (SUCCESSFUL_LOGIN_GROUP),
ADD (SERVER_ROLE_MEMBER_CHANGE_GROUP),
ADD (SERVER_PERMISSION_CHANGE_GROUP),
ADD (LOGIN_CHANGE_PASSWORD_GROUP),
ADD (DATABASE_CHANGE_GROUP)
WITH (STATE = ON);

-- Step 3: Database-level audit specification
CREATE DATABASE AUDIT SPECIFICATION [DBA_DBauditSpec]
FOR SERVER AUDIT [DBA_SecurityAudit]
ADD (SELECT ON dbo.Customers BY PUBLIC),
ADD (INSERT, UPDATE, DELETE ON dbo.Orders BY PUBLIC),
ADD (EXECUTE ON dbo.usp_TransferFunds BY PUBLIC),
ADD (DATABASE_PERMISSION_CHANGE_GROUP),
ADD (DATABASE_ROLE_MEMBER_CHANGE_GROUP)
WITH (STATE = ON);

-- Read audit log
SELECT *
FROM sys.fn_get_audit_file('D:\Audit\DBA_SecurityAudit*.sqlaudit', DEFAULT, DEFAULT)
ORDER BY event_time DESC;
```

### C2 Audit / Common Criteria Compliance
```sql
-- Enable C2 audit mode (all failed and successful logins + all SQL statements)
-- WARNING: High overhead; use targeted auditing instead
EXEC sp_configure 'c2 audit mode', 1;
RECONFIGURE;
```

---

## Row-Level Security (RLS)

```sql
-- Create security predicate function
CREATE FUNCTION dbo.fn_SecurityPredicate(@TenantId INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN SELECT 1 AS result
WHERE @TenantId = CAST(SESSION_CONTEXT(N'TenantId') AS INT)
   OR IS_MEMBER('db_owner') = 1;

-- Create security policy
CREATE SECURITY POLICY dbo.TenantSecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(TenantId)
    ON dbo.Orders,
ADD BLOCK PREDICATE dbo.fn_SecurityPredicate(TenantId)
    ON dbo.Orders AFTER INSERT
WITH (STATE = ON);

-- Set tenant context in application
EXEC sp_set_session_context N'TenantId', 5;
```

---

## Dynamic Data Masking

```sql
-- Mask email column (show only domain)
ALTER TABLE dbo.Customers
ALTER COLUMN Email ADD MASKED WITH (FUNCTION = 'email()');

-- Partial mask (show first/last characters)
ALTER TABLE dbo.Customers
ALTER COLUMN PhoneNumber ADD MASKED WITH (FUNCTION = 'partial(2,"XXX-XXX-",4)');

-- Default mask (replace with 0, xxxx, 01-01-1900, etc.)
ALTER TABLE dbo.Customers
ALTER COLUMN SSN ADD MASKED WITH (FUNCTION = 'default()');

-- Random number range mask
ALTER TABLE dbo.Salary
ALTER COLUMN Amount ADD MASKED WITH (FUNCTION = 'random(1, 100000)');

-- Grant UNMASK permission to privileged users
GRANT UNMASK TO [PrivilegedUser];
```

---

## Security Auditing Queries

### Privileged Access Review
```sql
-- All sysadmin members
SELECT m.name AS member_name, m.type_desc, m.is_disabled
FROM sys.server_role_members srm
JOIN sys.server_principals r ON srm.role_principal_id = r.principal_id
JOIN sys.server_principals m ON srm.member_principal_id = m.principal_id
WHERE r.name = 'sysadmin';

-- db_owner members in all databases
EXEC sp_MSforeachdb '
USE [?];
SELECT DB_NAME() AS db_name, dp.name AS member_name, dp.type_desc
FROM sys.database_role_members drm
JOIN sys.database_principals r ON drm.role_principal_id = r.principal_id
JOIN sys.database_principals dp ON drm.member_principal_id = dp.principal_id
WHERE r.name = ''db_owner'' AND dp.name NOT IN (''dbo'')';

-- Logins with dangerous permissions
SELECT
    sp.name AS login_name,
    srvp.permission_name,
    srvp.state_desc
FROM sys.server_permissions srvp
JOIN sys.server_principals sp ON srvp.grantee_principal_id = sp.principal_id
WHERE srvp.permission_name IN ('CONTROL SERVER', 'ALTER ANY LOGIN', 'ALTER ANY DATABASE')
ORDER BY sp.name;
```

### Security Configuration Checks
```sql
-- Check for SQL logins with weak policies
SELECT name, is_policy_checked, is_expiration_checked, is_disabled
FROM sys.sql_logins
WHERE is_policy_checked = 0 OR is_expiration_checked = 0;

-- Check linked servers (potential security risk)
SELECT name, product, provider, data_source,
       is_remote_login_enabled, is_rpc_out_enabled
FROM sys.servers WHERE is_linked = 1;

-- Check cross-db ownership chaining
SELECT name, is_db_chaining_on FROM sys.databases WHERE is_db_chaining_on = 1;

-- Check for TRUSTWORTHY databases
SELECT name, is_trustworthy_on FROM sys.databases WHERE is_trustworthy_on = 1;
-- TRUSTWORTHY should only be ON for msdb; disable on all user databases

-- Disable TRUSTWORTHY
ALTER DATABASE [YourDB] SET TRUSTWORTHY OFF;
```

---

## Best Practices Summary

1. **Least Privilege**: Grant minimum permissions required. Use custom roles.
2. **Disable SA**: Rename and disable the SA account; create named DBA accounts.
3. **Windows Authentication**: Prefer over SQL authentication where possible.
4. **Enable TDE**: For all production databases containing sensitive data.
5. **Audit logins**: Log all failed logins; consider logging successful logins.
6. **Disable xp_cmdshell**: Unless strictly required by a specific process.
7. **Disable CLR**: Unless using specific CLR assemblies; mark safe if needed.
8. **Disable OLE Automation**: Security risk; use alternatives.
9. **TRUSTWORTHY = OFF**: Default; do not enable unless absolutely required.
10. **Cross-DB Ownership Chaining**: Disable at instance and database level.
11. **Encrypt connections**: Force TLS for all connections.
12. **Patch regularly**: Stay within 1 year of current CU on supported versions.
13. **Remove PUBLIC permissions**: Audit what PUBLIC has access to.
14. **Contained databases**: Consider for cloud migration; use with care.
15. **Secrets management**: Never store credentials in SQL scripts or jobs; use Azure Key Vault, Windows Credential Manager, or secrets management APIs.

```sql
-- Security hardening: disable dangerous features
EXEC sp_configure 'xp_cmdshell', 0; RECONFIGURE;
EXEC sp_configure 'Ole Automation Procedures', 0; RECONFIGURE;
EXEC sp_configure 'clr enabled', 0; RECONFIGURE;
EXEC sp_configure 'remote admin connections', 0; RECONFIGURE;
EXEC sp_configure 'cross db ownership chaining', 0; RECONFIGURE;
```
