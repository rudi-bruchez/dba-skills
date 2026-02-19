# SQL Server Audit Configuration

---

## SQL Server Audit Architecture

SQL Server Audit (2008+) provides fine-grained auditing of instance and database activities.

```
Server Audit (defines WHERE to write)
├── Server Audit Specification (WHAT instance-level events)
└── Database Audit Specification (WHAT database-level events)
```

---

## Server Audit — Create and Configure

```sql
-- Write audit records to file
CREATE SERVER AUDIT [DBA_SecurityAudit]
TO FILE (
    FILEPATH           = N'D:\AuditLogs\',   -- Directory must exist; SQL account must have write access
    MAXSIZE            = 200 MB,              -- Max size per file before rolling
    MAX_ROLLOVER_FILES = 20,                  -- Keep up to 20 files; oldest deleted when full
    RESERVE_DISK_SPACE = OFF                  -- Do not pre-allocate disk space
)
WITH (
    QUEUE_DELAY  = 1000,    -- ms; 0 = synchronous (impacts performance); 1000 = async
    ON_FAILURE   = CONTINUE -- CONTINUE = write fails silently; SHUTDOWN = stops SQL Server if audit fails
);

ALTER SERVER AUDIT [DBA_SecurityAudit] WITH (STATE = ON);

-- Alternative: write to Windows Application Event Log
CREATE SERVER AUDIT [DBA_EventLogAudit]
TO APPLICATION_LOG
WITH (QUEUE_DELAY = 1000, ON_FAILURE = CONTINUE);

-- View all server audits
SELECT name, audit_file_path, is_state_enabled, on_failure_desc
FROM sys.server_audits;
```

---

## Server Audit Specification — Instance-Level Events

```sql
CREATE SERVER AUDIT SPECIFICATION [DBA_ServerAuditSpec]
FOR SERVER AUDIT [DBA_SecurityAudit]
ADD (FAILED_LOGIN_GROUP),                  -- All failed login attempts
ADD (SUCCESSFUL_LOGIN_GROUP),              -- All successful logins (verbose; use selectively)
ADD (LOGOUT_GROUP),                        -- Session logouts
ADD (SERVER_ROLE_MEMBER_CHANGE_GROUP),     -- sysadmin membership changes
ADD (SERVER_PERMISSION_CHANGE_GROUP),      -- Server-level GRANT/DENY/REVOKE
ADD (LOGIN_CHANGE_PASSWORD_GROUP),         -- Password changes
ADD (DATABASE_CHANGE_GROUP),              -- CREATE/ALTER/DROP DATABASE
ADD (SCHEMA_OBJECT_CHANGE_GROUP),         -- DDL on any database object
ADD (BACKUP_RESTORE_GROUP)                -- All backup and restore operations
WITH (STATE = ON);

-- Disable the specification (without dropping it)
ALTER SERVER AUDIT SPECIFICATION [DBA_ServerAuditSpec] WITH (STATE = OFF);

-- View all server audit specifications
SELECT name, server_specification_id, is_state_enabled
FROM sys.server_audit_specifications;

-- View individual action groups in a specification
SELECT a.name AS audit_name, sas.name AS spec_name, sasd.audit_action_name
FROM sys.server_audit_specification_details sasd
JOIN sys.server_audit_specifications sas ON sasd.server_specification_id = sas.server_specification_id
JOIN sys.server_audits a ON sas.audit_guid = a.audit_guid
ORDER BY sas.name, sasd.audit_action_name;
```

---

## Database Audit Specification — Database-Level Events

```sql
USE [YourDatabase];

CREATE DATABASE AUDIT SPECIFICATION [DBA_DBauditSpec]
FOR SERVER AUDIT [DBA_SecurityAudit]
ADD (SELECT ON dbo.Customers BY PUBLIC),               -- All SELECTs on Customers
ADD (INSERT, UPDATE, DELETE ON dbo.Orders BY PUBLIC),  -- DML on Orders
ADD (EXECUTE ON dbo.usp_TransferFunds BY PUBLIC),      -- Sensitive procedure calls
ADD (DATABASE_PERMISSION_CHANGE_GROUP),                -- GRANT/DENY/REVOKE in this DB
ADD (DATABASE_ROLE_MEMBER_CHANGE_GROUP),               -- Role membership changes
ADD (SCHEMA_OBJECT_PERMISSION_CHANGE_GROUP),           -- Object-level permission changes
ADD (USER_CHANGE_PASSWORD_GROUP),                      -- Contained user password changes
ADD (FAILED_DATABASE_AUTHENTICATION_GROUP)             -- Failed contained user logins
WITH (STATE = ON);

-- View database audit specifications
SELECT name, database_specification_id, is_state_enabled
FROM sys.database_audit_specifications;

-- View details
SELECT a.name AS audit_name, das.name AS spec_name,
       dasd.audit_action_name, dasd.object_name, dasd.principal_name
FROM sys.database_audit_specification_details dasd
JOIN sys.database_audit_specifications das
    ON dasd.database_specification_id = das.database_specification_id
JOIN sys.server_audits a ON das.audit_guid = a.audit_guid
ORDER BY dasd.audit_action_name;
```

---

## Reading Audit Logs

```sql
-- Read from a specific audit file
SELECT *
FROM sys.fn_get_audit_file('D:\AuditLogs\DBA_SecurityAudit*.sqlaudit', DEFAULT, DEFAULT)
ORDER BY event_time DESC;

-- Read from all audit files in a directory
SELECT
    event_time,
    action_id,
    succeeded,
    server_principal_name    AS login_name,
    database_name,
    object_name,
    statement,
    client_ip,
    application_name,
    additional_information
FROM sys.fn_get_audit_file('D:\AuditLogs\*.sqlaudit', DEFAULT, DEFAULT)
ORDER BY event_time DESC;

-- Common audit action_id values:
-- LG = Login, LO = Logout
-- SL = SELECT, IN = INSERT, UP = UPDATE, DL = DELETE, EX = EXECUTE
-- AL = ALTER, CR = CREATE, DR = DROP
-- LGFL = Failed Login

-- Filter: failed logins only
SELECT event_time, server_principal_name AS login_name, client_ip, additional_information
FROM sys.fn_get_audit_file('D:\AuditLogs\*.sqlaudit', DEFAULT, DEFAULT)
WHERE action_id = 'LGFL'
ORDER BY event_time DESC;

-- Filter: specific table access
SELECT event_time, server_principal_name, action_id, object_name, statement
FROM sys.fn_get_audit_file('D:\AuditLogs\*.sqlaudit', DEFAULT, DEFAULT)
WHERE object_name = 'Customers'
ORDER BY event_time DESC;
```

---

## Common Audit Action Groups Reference

| Group Name | What It Audits |
|---|---|
| `FAILED_LOGIN_GROUP` | All failed login attempts |
| `SUCCESSFUL_LOGIN_GROUP` | All successful logins |
| `SERVER_ROLE_MEMBER_CHANGE_GROUP` | Changes to fixed/user-defined server role membership |
| `SERVER_PERMISSION_CHANGE_GROUP` | GRANT/DENY/REVOKE at server level |
| `LOGIN_CHANGE_PASSWORD_GROUP` | Password changes for SQL logins |
| `DATABASE_CHANGE_GROUP` | CREATE/ALTER/DROP DATABASE |
| `BACKUP_RESTORE_GROUP` | All backup and restore operations |
| `DATABASE_PERMISSION_CHANGE_GROUP` | GRANT/DENY/REVOKE within a database |
| `DATABASE_ROLE_MEMBER_CHANGE_GROUP` | Database role membership changes |
| `SCHEMA_OBJECT_CHANGE_GROUP` | DDL (CREATE, ALTER, DROP) on schema objects |
| `SELECT` (object-level) | SELECT on a specific table/view |
| `INSERT`, `UPDATE`, `DELETE` (object-level) | DML on specific tables |
| `EXECUTE` (object-level) | Execution of specific stored procedures |

---

## Login Auditing via Configuration (Simple Alternative)

For basic failed login auditing without SQL Server Audit:

```sql
-- Instance-level: configure login auditing
-- 0 = None, 1 = Successful logins, 2 = Failed logins, 3 = Both
EXEC xp_instance_regwrite
    N'HKEY_LOCAL_MACHINE',
    N'Software\Microsoft\MSSQLServer\MSSQLServer',
    N'AuditLevel',
    REG_DWORD, 2;  -- Failed logins only

-- Restart SQL Server for this to take effect
-- Failed logins appear in the SQL Server error log:
EXEC sys.xp_readerrorlog 0, 1, N'Login failed', NULL, NULL, NULL, N'desc';
```

---

## Audit Maintenance

```sql
-- Check current audit file size and path
SELECT name, audit_file_path, is_state_enabled,
       max_file_size, max_rollover_files
FROM sys.server_audits;

-- Stop, modify, and restart an audit
ALTER SERVER AUDIT [DBA_SecurityAudit] WITH (STATE = OFF);
ALTER SERVER AUDIT [DBA_SecurityAudit]
TO FILE (FILEPATH = N'E:\AuditLogs\', MAXSIZE = 500 MB, MAX_ROLLOVER_FILES = 30);
ALTER SERVER AUDIT [DBA_SecurityAudit] WITH (STATE = ON);

-- Drop an audit (must disable first)
ALTER SERVER AUDIT SPECIFICATION [DBA_ServerAuditSpec] WITH (STATE = OFF);
DROP SERVER AUDIT SPECIFICATION [DBA_ServerAuditSpec];
ALTER SERVER AUDIT [DBA_SecurityAudit] WITH (STATE = OFF);
DROP SERVER AUDIT [DBA_SecurityAudit];
```
