# Permissions Model

SQL Server uses a hierarchical permissions model covering instance-level and database-level access.

## 1. Instance-Level Access (Logins)
Logins represent an identity at the server instance level.
- **Windows Authentication:** Prefer this for all users and services.
- **SQL Server Authentication:** Use only when Windows Authentication is not possible.
- **Fixed Server Roles:** `sysadmin`, `serveradmin`, `securityadmin`, etc. Grant these sparingly.
- **Server Permissions:** `VIEW ANY DEFINITION`, `CREATE ANY DATABASE`, etc.

## 2. Database-Level Access (Users)
Users represent an identity within a specific database.
- **Fixed Database Roles:** `db_owner`, `db_datareader`, `db_datawriter`, `db_denydatawriter`, etc.
- **Database Permissions:** `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `EXECUTE`, etc.
- **Object Permissions:** `SELECT ON Schema.Table`, `EXECUTE ON Schema.Procedure`, etc.

## 3. Best Practices (Least Privilege)
- **Use Database Roles:** Instead of granting permissions directly to users, create custom roles and grant permissions to them.
- **Grant Only What is Needed:** Avoid using `sysadmin` or `db_owner` for routine tasks.
- **Avoid Using the `sa` Account:** Rename or disable the `sa` account and create a separate administrative account with `sysadmin` privileges.
- **Renaming the `sa` Account:**
    ```sql
    ALTER LOGIN sa WITH NAME = [New_SA_Name];
    ```
- **Disabling the `sa` Account:**
    ```sql
    ALTER LOGIN sa DISABLE;
    ```
- **Auditing Permissions:** Regularly review the permissions granted to users and roles to ensure they are still necessary.

## 4. Querying Permissions
View the permissions granted to a user or role.
```sql
SELECT
    dp.name AS PrincipalName,
    dp.type_desc AS PrincipalType,
    o.name AS ObjectName,
    p.permission_name AS PermissionName,
    p.state_desc AS PermissionState
FROM sys.database_permissions AS p
JOIN sys.database_principals AS dp ON p.grantee_principal_id = dp.principal_id
LEFT JOIN sys.objects AS o ON p.major_id = o.object_id
WHERE dp.name = 'MyUserName';
```

## References
- [Microsoft Documentation: SQL Server Security - Permissions](https://learn.microsoft.com/en-us/sql/relational-databases/security/permissions-database-engine)
- [Microsoft Documentation: Fixed Database Roles](https://learn.microsoft.com/en-us/sql/relational-databases/security/authentication-access/database-level-roles)
- [Brent Ozar's SQL Server Security Best Practices (Checklist)](https://www.netwrix.com/sql_server_security_best_practices.html)
