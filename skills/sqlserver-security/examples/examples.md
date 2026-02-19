# SQL Server Security — Examples

---

## Scenario 1: Security Audit Findings — New Instance Review

**Context:** A security audit is requested for a newly inherited SQL Server instance before connecting it to the corporate network.

**Run the full audit checklist:**

```sql
-- 1. Authentication mode
SELECT SERVERPROPERTY('IsIntegratedSecurityOnly') AS windows_auth_only;
-- Result: 0 (Mixed mode enabled — flag for review if not needed)

-- 2. SA account status
SELECT name, is_disabled FROM sys.sql_logins WHERE name IN ('sa', 'SA');
-- Result: sa, is_disabled = 0 (CRITICAL: SA is enabled)

-- 3. Sysadmin members
SELECT m.name, m.type_desc, m.is_disabled
FROM sys.server_role_members srm
JOIN sys.server_principals r ON srm.role_principal_id = r.principal_id
JOIN sys.server_principals m ON srm.member_principal_id = m.principal_id
WHERE r.name = 'sysadmin';
-- Result: sa, AppUser, BUILTIN\Administrators (CRITICAL: AppUser should not be sysadmin)

-- 4. Dangerous features enabled
SELECT name, value_in_use FROM sys.configurations
WHERE name IN ('xp_cmdshell', 'Ole Automation Procedures', 'clr enabled');
-- Result: xp_cmdshell = 1 (HIGH: should be 0)

-- 5. TRUSTWORTHY databases
SELECT name FROM sys.databases WHERE is_trustworthy_on = 1 AND name <> 'msdb';
-- Result: SalesDB (MEDIUM: should be OFF)

-- 6. SQL logins without policy enforcement
SELECT name, is_policy_checked, is_expiration_checked FROM sys.sql_logins
WHERE is_policy_checked = 0 OR is_expiration_checked = 0;
-- Result: ReportUser (MEDIUM: no policy, no expiration)
```

**Remediation steps:**
```sql
-- Fix SA
ALTER LOGIN sa DISABLE;

-- Remove AppUser from sysadmin
ALTER SERVER ROLE [sysadmin] DROP MEMBER [AppUser];

-- Disable xp_cmdshell
EXEC sp_configure 'xp_cmdshell', 0; RECONFIGURE;
EXEC sp_configure 'Ole Automation Procedures', 0; RECONFIGURE;

-- Turn off TRUSTWORTHY
ALTER DATABASE [SalesDB] SET TRUSTWORTHY OFF;

-- Fix policy on SQL login
ALTER LOGIN [ReportUser] WITH CHECK_POLICY = ON, CHECK_EXPIRATION = ON;
```

---

## Scenario 2: Fixing Orphaned Users After Database Migration

**Context:** The `InventoryDB` database was restored from a backup to a new server. Several applications are failing to connect because their database users no longer have matching logins on the new instance.

**Step 1 — Find orphaned users**
```sql
USE [InventoryDB];
SELECT dp.name AS user_name, dp.type_desc, dp.create_date
FROM sys.database_principals dp
LEFT JOIN sys.server_principals sp ON dp.sid = sp.sid
WHERE dp.type IN ('S', 'U')
  AND dp.name NOT IN ('dbo', 'guest', 'INFORMATION_SCHEMA', 'sys')
  AND sp.name IS NULL;
-- Result:
-- InventoryApp   SQL_USER   (orphaned — no matching login on new server)
-- ReportService  SQL_USER   (orphaned)
-- CORP\OldUser   WINDOWS_USER (orphaned — user left company)
```

**Step 2 — Create matching logins on new server (for SQL logins)**
```sql
-- Create the SQL Server login (use a strong password; rotate after setup)
CREATE LOGIN [InventoryApp]
WITH PASSWORD = 'TempPassword_ChangeMe!',
     CHECK_POLICY = ON, CHECK_EXPIRATION = ON;

-- Map the existing database user to the new login
-- This preserves all existing permissions without reassigning them
ALTER USER [InventoryApp] WITH LOGIN = [InventoryApp];
```

**Step 3 — Map the Windows login (if the account exists in AD on new domain)**
```sql
-- Create the Windows login on new server
CREATE LOGIN [NEWCORP\ReportServiceAccount] FROM WINDOWS;

-- Map the database user to the new login by SID matching is not possible (different domain)
-- Drop old user and recreate
DROP USER [ReportService];
CREATE USER [ReportService] FOR LOGIN [NEWCORP\ReportServiceAccount];

-- Restore permissions (get from original server or re-grant)
ALTER ROLE [db_datareader] ADD MEMBER [ReportService];
GRANT EXECUTE ON SCHEMA::dbo TO [ReportService];
```

**Step 4 — Remove orphaned user for departed employee**
```sql
-- Remove the departed employee's user
DROP USER [CORP\OldUser];
-- If they own objects, transfer ownership first:
-- ALTER AUTHORIZATION ON SCHEMA::PersonalSchema TO dbo;
-- ALTER AUTHORIZATION ON dbo.OldUserTable TO dbo;
```

**Step 5 — Verify no more orphaned users**
```sql
SELECT dp.name
FROM sys.database_principals dp
LEFT JOIN sys.server_principals sp ON dp.sid = sp.sid
WHERE dp.type IN ('S', 'U')
  AND dp.name NOT IN ('dbo', 'guest', 'INFORMATION_SCHEMA', 'sys')
  AND sp.name IS NULL;
-- Empty result set = all users have matching logins
```

---

## Scenario 3: Enabling TDE on a Production Database

**Context:** A compliance audit requires all production databases containing PII to be encrypted at rest by end of quarter. The target database is `CustomerDB` (80 GB).

**Pre-requisites check:**
```sql
-- Verify master key exists in master database
SELECT name FROM sys.symmetric_keys WHERE name = '##MS_DatabaseMasterKey##';
-- If empty: CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'StrongMasterKey!';

-- Check if CustomerDB is already encrypted
SELECT DB_NAME(database_id) AS db_name, encryption_state_desc
FROM sys.dm_database_encryption_keys;
-- Not present = not encrypted
```

**Implementation:**
```sql
-- Step 1: Create TDE certificate (run in master)
USE master;
CREATE CERTIFICATE TDECert_CustomerDB WITH SUBJECT = 'TDE for CustomerDB';

-- Step 2: Back up the certificate FIRST (critical — do not skip)
BACKUP CERTIFICATE TDECert_CustomerDB
TO FILE = 'D:\CertBackups\TDECert_CustomerDB.cer'
WITH PRIVATE KEY (
    FILE = 'D:\CertBackups\TDECert_CustomerDB.pvk',
    ENCRYPTION BY PASSWORD = 'CertPvkSecret2024!'
);
-- Confirm the backup files exist and copy to secure offsite location

-- Step 3: Create DEK in the target database
USE [CustomerDB];
CREATE DATABASE ENCRYPTION KEY
WITH ALGORITHM = AES_256
ENCRYPTION BY SERVER CERTIFICATE TDECert_CustomerDB;

-- Step 4: Enable TDE
ALTER DATABASE [CustomerDB] SET ENCRYPTION ON;

-- Step 5: Monitor progress (can query repeatedly)
SELECT DB_NAME(database_id), encryption_state_desc, percent_complete
FROM sys.dm_database_encryption_keys;
-- Phase: encryption_state = 2 (in progress), then 3 (complete)
-- 80 GB: expect 10-60 minutes depending on disk speed; no downtime required
```

**Post-encryption verification:**
```sql
SELECT DB_NAME(database_id) AS database_name,
       encryption_state_desc,
       encryptor_thumbprint,
       key_algorithm,
       key_length
FROM sys.dm_database_encryption_keys
WHERE DB_NAME(database_id) = 'CustomerDB';
-- encryption_state_desc = Encrypted
-- key_algorithm = AES, key_length = 256

-- Verify backup files are also encrypted (they inherit TDE automatically)
-- Run a test backup and confirm it cannot be attached to another server without the certificate
BACKUP DATABASE [CustomerDB]
TO DISK = N'D:\Backups\CustomerDB_TDE_Test.bak'
WITH COMPRESSION, CHECKSUM, STATS = 10;
```

---

## Scenario 4: Configuring Login Auditing and Detecting Suspicious Activity

**Context:** The security team wants to know who is logging in to a sensitive server, with immediate alerting on failed logins.

**Step 1 — Enable SQL Server Audit for login events**
```sql
-- Create server audit
CREATE SERVER AUDIT [LoginAudit]
TO FILE (
    FILEPATH           = N'D:\AuditLogs\',
    MAXSIZE            = 500 MB,
    MAX_ROLLOVER_FILES = 30
)
WITH (QUEUE_DELAY = 1000, ON_FAILURE = CONTINUE);

ALTER SERVER AUDIT [LoginAudit] WITH (STATE = ON);

-- Create server audit specification (logins only)
CREATE SERVER AUDIT SPECIFICATION [LoginAuditSpec]
FOR SERVER AUDIT [LoginAudit]
ADD (FAILED_LOGIN_GROUP),
ADD (SUCCESSFUL_LOGIN_GROUP),
ADD (SERVER_ROLE_MEMBER_CHANGE_GROUP),
ADD (LOGIN_CHANGE_PASSWORD_GROUP)
WITH (STATE = ON);
```

**Step 2 — Query the audit log for suspicious patterns**
```sql
-- Failed logins in the last 24 hours
SELECT event_time, server_principal_name AS login_name, client_ip,
       additional_information, succeeded
FROM sys.fn_get_audit_file('D:\AuditLogs\LoginAudit*.sqlaudit', DEFAULT, DEFAULT)
WHERE action_id = 'LGFL'  -- Failed login
  AND event_time >= DATEADD(HOUR, -24, GETUTCDATE())
ORDER BY event_time DESC;

-- Brute force detection: > 10 failed logins from same IP in 1 hour
SELECT
    client_ip,
    server_principal_name AS login_name,
    COUNT(*) AS failed_attempts,
    MIN(event_time) AS first_attempt,
    MAX(event_time) AS last_attempt
FROM sys.fn_get_audit_file('D:\AuditLogs\LoginAudit*.sqlaudit', DEFAULT, DEFAULT)
WHERE action_id = 'LGFL'
  AND event_time >= DATEADD(HOUR, -1, GETUTCDATE())
GROUP BY client_ip, server_principal_name
HAVING COUNT(*) > 10
ORDER BY failed_attempts DESC;

-- After-hours logins (outside 6 AM - 10 PM)
SELECT event_time, server_principal_name AS login_name, client_ip, application_name
FROM sys.fn_get_audit_file('D:\AuditLogs\LoginAudit*.sqlaudit', DEFAULT, DEFAULT)
WHERE action_id = 'LG'  -- Successful login
  AND DATEPART(HOUR, event_time) NOT BETWEEN 6 AND 22
  AND event_time >= DATEADD(DAY, -7, GETUTCDATE())
ORDER BY event_time DESC;
```

**Step 3 — Respond to detected brute force (example: IP 192.168.100.45, 847 failed attempts)**
```sql
-- Identify which logins are being attacked
SELECT server_principal_name, COUNT(*) AS attempts
FROM sys.fn_get_audit_file('D:\AuditLogs\LoginAudit*.sqlaudit', DEFAULT, DEFAULT)
WHERE action_id = 'LGFL' AND client_ip = '192.168.100.45'
GROUP BY server_principal_name
ORDER BY attempts DESC;
-- Results: sa (520 attempts), admin (200 attempts), Administrator (127 attempts)

-- Verify SA is already disabled
SELECT name, is_disabled FROM sys.sql_logins WHERE name = 'sa';
-- is_disabled = 1 (good — attacks are futile)

-- Block at the firewall/network level (outside SQL Server)
-- Also notify the security team; the source IP may indicate an internal threat
```
