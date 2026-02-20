# PostgreSQL Security Examples

Practical scenarios and how to implement security best practices in PostgreSQL.

## Scenario 1: Implementing Least Privilege for a Reporting Application

### Diagnosis
1.  **Issue:** A reporting application is currently connecting as the `postgres` superuser in the `mydb` database.
2.  **Goal:** Reduce the application's permissions to the minimum necessary for its reporting tasks.

### Procedure
1.  **Create a Reporting Role:**
```sql
CREATE ROLE reporting_user WITH LOGIN PASSWORD 'StrongPassword123!';
```
2.  **Grant Select Permissions to the Role:**
```sql
GRANT SELECT ON ALL TABLES IN SCHEMA public TO reporting_user;
```
3.  **Verify the Permissions:**
```sql
SELECT usename, usecreatedb, usesuper, usecatupd, userepl
FROM pg_user
WHERE usename = 'reporting_user';
```
**Result:** The application now only has `SELECT` permissions on all tables in the `public` schema, fulfilling its reporting requirements while minimizing security risk.

## Scenario 2: Enforcing SCRAM-SHA-256 Authentication

### Diagnosis
1.  **Goal:** Enforce strong `scram-sha-256` password authentication for all PostgreSQL users.

### Procedure
1.  **Update `pg_hba.conf`:**
    Set the method to `scram-sha-256` for all connections.
    ```conf
    # Enforce SCRAM-SHA-256 for all IPv4 connections
    host    all             all             0.0.0.0/0               scram-sha-256
    ```
2.  **Reload the Configuration:**
    ```bash
    sudo systemctl reload postgresql
    ```
3.  **Change User Passwords:**
    Ensure all users have strong passwords and that their authentication method is `scram-sha-256`.
    ```sql
    ALTER ROLE myuser WITH PASSWORD 'StrongPassword123!';
    ```
**Result:** All users must now use `scram-sha-256` to authenticate, providing more secure password protection.

## Scenario 3: Protecting Sensitive Data with RLS

### Diagnosis
1.  **Goal:** Restrict users from seeing rows in the `orders` table that don't belong to their specific region.

### Procedure
1.  **Enable RLS on the `orders` table:**
```sql
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
```
2.  **Create a Security Policy:**
```sql
CREATE POLICY sales_region_policy ON orders
FOR SELECT
TO public
USING (region = current_user OR current_user = 'manager'); -- Manager sees all
```
**Result:** Users will now only see rows where the `region` column matches their database username, or if they are the `manager` user.
```sql
-- Test for a specific user
SET ROLE west_region_user;
SELECT * FROM orders; -- Only shows West region rows
RESET ROLE;
```

## Scenario 4: Auditing DDL and DML Operations with pgaudit

### Diagnosis
1.  **Goal:** Track all `CREATE`, `ALTER`, and `DROP` (DDL) and `INSERT`, `UPDATE`, and `DELETE` (DML) operations in the `mydb` database.

### Procedure
1.  **Install the `pgaudit` Extension:**
    ```bash
    sudo apt-get install postgresql-15-pgaudit
    ```
2.  **Configure `postgresql.conf`:**
    Add `pgaudit` to `shared_preload_libraries`.
    ```conf
    shared_preload_libraries = 'pgaudit'
    ```
3.  **Reload the Configuration:**
    ```bash
    sudo systemctl restart postgresql
    ```
4.  **Configure pgaudit in the Database:**
    Set the log level to `ddl` and `write`.
    ```sql
    SET pgaudit.log = 'ddl, write';
    ```
**Result:** All DDL and DML operations are now logged to the PostgreSQL log files, providing a detailed audit trail of database activity.
