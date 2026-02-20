# Roles & Privileges

PostgreSQL uses a role-based access control (RBAC) model to manage authentication and authorization.

## 1. Roles and Groups
Roles can be users (with login privileges) or groups (without login privileges).
- **Create a role:** Use `CREATE ROLE [role_name]`.
- **Grant permissions to a role:** Use `GRANT [privilege] ON [object] TO [role_name]`.
- **Assign a role to another role:** Use `GRANT [role_name] TO [user_name]`.

## 2. Fixed Database Roles
- **`public`:** Automatically granted to all users. Revoke unnecessary privileges from this role.
- **`db_owner` (if applicable):** Typically managed by a specific role or group.
- **`postgres`:** The default superuser. Use it only for routine administrative tasks.

## 3. Best Practices (Least Privilege)
- **Grant Only What is Needed:** Avoid using `postgres` or `db_owner` for routine tasks.
- **Use Group Roles:** Grant permissions to group roles and then assign users to those groups.
- **Avoid Using the `postgres` Account:** Rename or disable the `postgres` account and create a separate administrative account with `superuser` privileges.
- **Renaming the `postgres` Account:**
    ```sql
    ALTER ROLE postgres WITH NAME = [New_Postgres_Name];
    ```
- **Disabling the `postgres` Account:**
    ```sql
    ALTER ROLE postgres WITH NOLOGIN;
    ```
- **Auditing Privileges:** Regularly review the privileges granted to roles to ensure they are still necessary.

## 4. Querying Privileges
View the privileges granted to a role.
```sql
SELECT
    usename,
    usecreatedb,
    usesuper,
    usecatupd,
    userepl
FROM pg_user
WHERE usename = 'my_user_name';
```

## References
- [PostgreSQL Documentation: Database Roles](https://www.postgresql.org/docs/current/user-manag.html)
- [PostgreSQL Documentation: GRANT](https://www.postgresql.org/docs/current/sql-grant.html)
- [PostgreSQL Documentation: REVOKE](https://www.postgresql.org/docs/current/sql-revoke.html)
- [pganalyze: Guide to PostgreSQL Roles and Privileges](https://pganalyze.com/blog/postgresql-roles-privileges)
---
