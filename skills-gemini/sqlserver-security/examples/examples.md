# SQL Server Security Examples

Practical scenarios and how to implement security best practices in SQL Server.

## Scenario 1: Implementing Least Privilege for a Reporting Application

### Diagnosis
1.  **Issue:** A reporting application is currently connecting as a `db_owner` in the `Sales` database.
2.  **Goal:** Reduce the application's permissions to the minimum necessary for its reporting tasks.

### Procedure
1.  **Create a Reporting Role:**
```sql
USE [Sales];
CREATE ROLE [ReportingRole];
```
2.  **Grant Select Permissions to the Role:**
```sql
GRANT SELECT ON SCHEMA::dbo TO [ReportingRole];
```
3.  **Assign the Application User to the Role:**
```sql
ALTER ROLE [ReportingRole] ADD MEMBER [ReportingAppUser];
```
4.  **Verify the Permissions:**
```sql
SELECT
    dp.name AS PrincipalName,
    p.permission_name AS PermissionName,
    o.name AS ObjectName
FROM sys.database_permissions AS p
JOIN sys.database_principals AS dp ON p.grantee_principal_id = dp.principal_id
LEFT JOIN sys.objects AS o ON p.major_id = o.object_id
WHERE dp.name = 'ReportingRole';
```
**Result:** The application now only has `SELECT` permissions on all tables and views in the `dbo` schema, fulfilling its reporting requirements while minimizing security risk.

## Scenario 2: Enforcing Password Policies and MFA

### Diagnosis
1.  **Goal:** Enforce strong password policies and MFA for all SQL Server administrative accounts.

### Procedure
1.  **Enable Entra ID (formerly Azure AD) Integration:** (If using Azure or a hybrid environment)
2.  **Configure Entra ID for SQL Server:** Follow the Microsoft documentation for integrating SQL Server with Entra ID.
3.  **Enforce MFA for Entra ID Accounts:** Use Azure AD Conditional Access policies to enforce MFA for all administrative logins.
4.  **For On-Premises SQL Server:**
    - Use Windows Authentication and Active Directory for all user accounts.
    - Enforce strong password policies and MFA via Active Directory.
    - Limit the number of administrative accounts and regularly review their permissions.

## Scenario 3: Protecting Sensitive Data with TDE

### Diagnosis
1.  **Goal:** Encrypt the entire `CustomerData` database to protect sensitive customer information at rest.

### Procedure
1.  **Create a Master Key:**
```sql
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'StrongPassword123!';
```
2.  **Create a Certificate for TDE:**
```sql
CREATE CERTIFICATE TDE_Cert WITH SUBJECT = 'TDE Encryption Certificate';
```
3.  **Create a Database Encryption Key (DEK):**
```sql
USE [CustomerData];
CREATE DATABASE ENCRYPTION KEY WITH ALGORITHM = AES_256 ENCRYPTION BY SERVER CERTIFICATE TDE_Cert;
```
4.  **Enable Encryption on the Database:**
```sql
ALTER DATABASE [CustomerData] SET ENCRYPTION ON;
```
**Result:** The entire `CustomerData` database is now encrypted at rest, including its data files, log files, and backups.

## Scenario 4: Implementing Row-Level Security (RLS)

### Diagnosis
1.  **Goal:** Restrict users from seeing rows in the `Sales` table that don't belong to their specific region.

### Procedure
1.  **Create a Security Schema:**
```sql
CREATE SCHEMA [Security];
```
2.  **Create a Filter Function:**
```sql
CREATE FUNCTION [Security].[fn_SalesFilter](@Region AS sysname)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN SELECT 1 AS result
WHERE @Region = USER_NAME() OR USER_NAME() = 'Manager'; -- Manager sees all
```
3.  **Create a Security Policy:**
```sql
CREATE SECURITY POLICY [Security].[SalesPolicy]
ADD FILTER PREDICATE [Security].[fn_SalesFilter](Region) ON dbo.Sales
WITH (STATE = ON);
```
**Result:** Users will now only see rows where the `Region` column matches their database username, or if they are the `Manager` user.
```sql
-- Test for a specific user
EXECUTE AS USER = 'WestRegionUser';
SELECT * FROM dbo.Sales; -- Only shows West region rows
REVERT;
```
