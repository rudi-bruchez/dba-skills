# RLS Guide

Row-Level Security (RLS) allows you to restrict data access at the row level based on a user's identity or role.

## 1. Requirements

- **PostgreSQL 9.5+:** RLS was introduced in version 9.5.
- **Enable RLS on a table:** Use `ALTER TABLE [table] ENABLE ROW LEVEL SECURITY;`.
- **Create a policy:** Define a policy that specifies which rows a user can see.

## 2. Basic Example

Suppose you have a `Sales` table with a `Region` column and want users to see only their own rows.

1.  **Enable RLS on the table:**
    ```sql
    ALTER TABLE Sales ENABLE ROW LEVEL SECURITY;
    ```
2.  **Create a filter function:** (Optional, but recommended for reusable logic)
    ```sql
    CREATE FUNCTION check_sales_access(region text)
    RETURNS boolean
    AS $$
    BEGIN
        RETURN (region = current_user OR current_user = 'manager');
    END;
    $$ LANGUAGE plpgsql SECURITY DEFINER;
    ```
3.  **Create a security policy:**
    ```sql
    CREATE POLICY sales_access_policy ON Sales
    FOR SELECT
    TO public
    USING (check_sales_access(Region));
    ```
    - **USING:** Defines the expression that determines which rows are visible.

## 3. Best Practices

- **Use Functions for Reusable Logic:** Keep policy logic centralized and easy to maintain.
- **Grant Permissions to Roles:** Grant only the necessary permissions to roles to manage data access.
- **Monitor RLS Policies:** Regularly review RLS policies to ensure they are still correct and necessary.
- **Be Aware of Performance Impact:** Complex RLS policies can affect query performance. Test and optimize if needed.

## References
- [PostgreSQL Documentation: Row-Level Security](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)
- [PostgreSQL Documentation: CREATE POLICY](https://www.postgresql.org/docs/current/sql-createpolicy.html)
- [pganalyze: Guide to PostgreSQL Row-Level Security](https://pganalyze.com/blog/postgresql-row-level-security)
