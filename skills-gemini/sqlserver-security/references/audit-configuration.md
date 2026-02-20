# Audit Configuration

SQL Server Audit provides a built-in mechanism for tracking server-level and database-level security events.

## 1. Create a Server Audit
This object defines the destination (e.g., file, application log, or security log) for the audit data.
```sql
CREATE SERVER AUDIT [MyServerAudit]
TO FILE ( FILEPATH = 'C:\AuditLogs' );
-- Enable the audit
ALTER SERVER AUDIT [MyServerAudit] WITH (STATE = ON);
```

## 2. Create a Server Audit Specification
Tracks server-level events like failed logins and role changes.
```sql
CREATE SERVER AUDIT SPECIFICATION [MyServerAuditSpec]
FOR SERVER AUDIT [MyServerAudit]
ADD (FAILED_LOGIN_GROUP),
ADD (SERVER_ROLE_MEMBER_CHANGE_GROUP),
ADD (DATABASE_CHANGE_GROUP);
-- Enable the specification
ALTER SERVER AUDIT SPECIFICATION [MyServerAuditSpec] WITH (STATE = ON);
```

## 3. Create a Database Audit Specification
Tracks database-level events like schema changes and data access.
```sql
USE [MyDatabase];
CREATE DATABASE AUDIT SPECIFICATION [MyDatabaseAuditSpec]
FOR SERVER AUDIT [MyServerAudit]
ADD (SCHEMA_OBJECT_CHANGE_GROUP),
ADD (DATABASE_OBJECT_PERMISSION_CHANGE_GROUP),
ADD (SELECT ON SCHEMA::dbo BY [public]);
-- Enable the specification
ALTER DATABASE AUDIT SPECIFICATION [MyDatabaseAuditSpec] WITH (STATE = ON);
```

## 4. Query Audit Logs
View the captured audit data using the `sys.fn_get_audit_file` function.
```sql
SELECT
    event_time,
    action_id,
    server_principal_name,
    database_name,
    object_name,
    statement
FROM sys.fn_get_audit_file('C:\AuditLogs\*', DEFAULT, DEFAULT)
ORDER BY event_time DESC;
```

## Best Practices
- **Destination:** Store audit files in a secure location with limited access.
- **Retention:** Define a retention policy to manage the volume of audit data.
- **Monitoring:** Regularly review the audit logs for suspicious activity.
- **MFA:** Enforce MFA for administrative accounts that can modify audit settings.

## References
- [Microsoft Documentation: SQL Server Audit](https://learn.microsoft.com/en-us/sql/relational-databases/security/auditing/sql-server-audit-database-engine)
- [Brent Ozar's Guide to SQL Server Audit](https://www.brentozar.com/sql/auditing/)
