# SQL Server Permissions Model

---

## Principal Hierarchy

```
Server Level (instance)
├── Logins
│   ├── SQL Server login (username + password stored in SQL Server)
│   ├── Windows login (maps to AD user or service account)
│   └── Windows group login (maps to AD security group)
└── Server Roles
    ├── Fixed server roles (sysadmin, serveradmin, securityadmin, etc.)
    └── User-defined server roles (SQL 2012+)

Database Level
├── Database Users (mapped from logins, or contained users)
└── Database Roles
    ├── Fixed database roles (db_owner, db_datareader, etc.)
    └── User-defined database roles

Schema Level
└── Schemas (containers for objects; grant at schema level to cover all objects)

Object Level
└── Tables, views, stored procedures, functions, sequences, etc.
```

---

## Creating Logins (Instance Level)

```sql
-- SQL Server login (mixed mode only)
CREATE LOGIN [AppUser]
WITH PASSWORD    = 'StrongPass123!',
     CHECK_POLICY     = ON,   -- Enforce Windows password policy
     CHECK_EXPIRATION  = ON,   -- Enforce password expiration
     DEFAULT_DATABASE = [YourDB];

-- Windows login
CREATE LOGIN [DOMAIN\AppServiceAccount] FROM WINDOWS;

-- Windows group login (all group members inherit access)
CREATE LOGIN [DOMAIN\DBATeam] FROM WINDOWS WITH DEFAULT_DATABASE = [master];

-- Disable a login
ALTER LOGIN [AppUser] DISABLE;

-- Unlock a locked account
ALTER LOGIN [AppUser] WITH PASSWORD = 'NewPass123!' UNLOCK;

-- List all logins
SELECT name, type_desc, is_disabled, is_policy_checked,
       is_expiration_checked, default_database_name, create_date
FROM sys.server_principals
WHERE type IN ('S', 'U', 'G')
ORDER BY type_desc, name;
```

---

## Fixed Server Roles

| Role | Permissions | Use Sparingly? |
|---|---|---|
| `sysadmin` | Full control of SQL Server instance | Yes — named human DBA accounts only |
| `serveradmin` | Server configuration, SHUTDOWN | Yes |
| `securityadmin` | Manage logins, grant server permissions (can escalate to sysadmin!) | Yes |
| `processadmin` | Kill connections | Moderate |
| `dbcreator` | CREATE, ALTER, DROP databases | Yes |
| `bulkadmin` | BULK INSERT | Limited use |
| `diskadmin` | Manage disk files | Rarely needed |
| `public` | Default role — all logins are members | Cannot be removed |

```sql
-- Add login to server role
ALTER SERVER ROLE [sysadmin] ADD MEMBER [DOMAIN\DBAAccount];
ALTER SERVER ROLE [dbcreator] ADD MEMBER [DOMAIN\DevTeam];

-- Remove from server role
ALTER SERVER ROLE [sysadmin] DROP MEMBER [OldDBAAccount];

-- List server role members
SELECT r.name AS role_name, m.name AS member_name, m.type_desc, m.is_disabled
FROM sys.server_role_members srm
JOIN sys.server_principals r ON srm.role_principal_id = r.principal_id
JOIN sys.server_principals m ON srm.member_principal_id = m.principal_id
ORDER BY r.name, m.name;
```

---

## Database Users

```sql
-- Create a user from a login
USE [YourDatabase];
CREATE USER [AppUser] FOR LOGIN [AppUser] WITH DEFAULT_SCHEMA = dbo;

-- Create a user from a Windows login
CREATE USER [AppService] FOR LOGIN [DOMAIN\AppServiceAccount];

-- Contained database user (SQL 2012+ — no server login required)
CREATE USER [ContainedUser] WITH PASSWORD = 'StrongPass123!';

-- Orphaned user check
SELECT dp.name AS user_name, dp.type_desc
FROM sys.database_principals dp
LEFT JOIN sys.server_principals sp ON dp.sid = sp.sid
WHERE dp.type IN ('S', 'U')
  AND dp.name NOT IN ('dbo', 'guest', 'INFORMATION_SCHEMA', 'sys')
  AND sp.name IS NULL;

-- Fix orphaned user
ALTER USER [OrphanedUser] WITH LOGIN = [MatchingLogin];

-- Drop a user (only if they own no objects)
DROP USER [OldUser];
```

---

## Fixed Database Roles

| Role | Permissions |
|---|---|
| `db_owner` | Full control of the database |
| `db_datareader` | SELECT on all tables and views |
| `db_datawriter` | INSERT, UPDATE, DELETE on all tables |
| `db_ddladmin` | CREATE, ALTER, DROP objects (no permissions changes) |
| `db_securityadmin` | Manage database roles and permissions |
| `db_backupoperator` | BACKUP DATABASE, BACKUP LOG |
| `db_denydatareader` | DENY SELECT on all tables (overrides granted permissions) |
| `db_denydatawriter` | DENY DML on all tables |

```sql
-- Add user to database role
ALTER ROLE [db_datareader] ADD MEMBER [AppUser];
ALTER ROLE [db_datawriter] ADD MEMBER [AppUser];

-- Remove from role
ALTER ROLE [db_datareader] DROP MEMBER [AppUser];

-- List database role members
SELECT r.name AS role_name, m.name AS member_name, m.type_desc
FROM sys.database_role_members drm
JOIN sys.database_principals r ON drm.role_principal_id = r.principal_id
JOIN sys.database_principals m ON drm.member_principal_id = m.principal_id
ORDER BY r.name, m.name;
```

---

## GRANT / DENY / REVOKE Syntax

```sql
-- Object-level permissions
GRANT SELECT ON dbo.Customers TO [AppUser];
GRANT INSERT, UPDATE ON dbo.Orders TO [AppUser];
GRANT EXECUTE ON dbo.usp_GetOrderStatus TO [AppUser];
DENY DELETE ON dbo.Customers TO [AppUser];
REVOKE SELECT ON dbo.Customers FROM [AppUser];

-- Schema-level permissions (applies to all objects currently and in future)
GRANT SELECT ON SCHEMA::Reporting TO [ReportUser];
GRANT EXECUTE ON SCHEMA::dbo TO [AppUser];
DENY SELECT ON SCHEMA::Finance TO [AppUser];  -- Prevent access to entire schema

-- Database-level permissions
GRANT VIEW DATABASE STATE TO [MonitorUser];  -- View DMVs
GRANT VIEW DEFINITION TO [DevUser];          -- View object definitions
GRANT CREATE TABLE TO [DevUser];
GRANT ALTER ANY SCHEMA TO [DevUser];

-- Instance-level permissions
GRANT VIEW SERVER STATE TO [MonitorLogin];   -- View DMVs
GRANT ALTER TRACE TO [DBALogin];

-- WITH GRANT OPTION (allows grantee to grant to others — use carefully)
GRANT SELECT ON dbo.Products TO [AppUser] WITH GRANT OPTION;
```

---

## Custom Database Roles (Least-Privilege Pattern)

Never use `db_datareader` / `db_datawriter` directly. Create custom roles scoped to exactly what the application needs.

```sql
-- Application role: read orders and customers, execute order procedures
CREATE ROLE [Role_OrdersApp];
GRANT SELECT  ON dbo.Orders    TO [Role_OrdersApp];
GRANT SELECT  ON dbo.Customers TO [Role_OrdersApp];
GRANT INSERT  ON dbo.Orders    TO [Role_OrdersApp];
GRANT UPDATE  ON dbo.Orders    TO [Role_OrdersApp];
GRANT EXECUTE ON dbo.usp_PlaceOrder   TO [Role_OrdersApp];
GRANT EXECUTE ON dbo.usp_CancelOrder  TO [Role_OrdersApp];

-- Add the application service account to the role
ALTER ROLE [Role_OrdersApp] ADD MEMBER [AppServiceUser];

-- Report reader role
CREATE ROLE [Role_ReportReader];
GRANT SELECT ON SCHEMA::Reporting TO [Role_ReportReader];

-- Add all report users
ALTER ROLE [Role_ReportReader] ADD MEMBER [ReportUser1];
ALTER ROLE [Role_ReportReader] ADD MEMBER [ReportUser2];
```

---

## Permission Reporting Queries

```sql
-- All database-level permissions
SELECT
    dp.name           AS principal_name,
    dp.type_desc      AS principal_type,
    o.name            AS object_name,
    o.type_desc       AS object_type,
    p.permission_name,
    p.state_desc      AS permission_state
FROM sys.database_permissions p
LEFT JOIN sys.database_principals dp ON p.grantee_principal_id = dp.principal_id
LEFT JOIN sys.objects o              ON p.major_id = o.object_id
ORDER BY dp.name, o.name, p.permission_name;

-- Effective permissions for a specific user (impersonation)
EXECUTE AS USER = 'AppUser';
SELECT * FROM fn_my_permissions(NULL, 'DATABASE');
REVERT;

-- Server-level permissions
SELECT sp.name AS login_name, sp.type_desc, srvp.permission_name, srvp.state_desc
FROM sys.server_permissions srvp
JOIN sys.server_principals sp ON srvp.grantee_principal_id = sp.principal_id
ORDER BY sp.name, srvp.permission_name;

-- PUBLIC role permissions (what every login can do by default)
SELECT o.name AS object_name, p.permission_name, p.state_desc
FROM sys.database_permissions p
JOIN sys.database_principals dp ON p.grantee_principal_id = dp.principal_id
LEFT JOIN sys.objects o ON p.major_id = o.object_id
WHERE dp.name = 'public'
ORDER BY o.name;
```
