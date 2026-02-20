# Encryption Guide

SQL Server provides several options for encrypting sensitive data at rest and in transit.

## 1. Data in Transit (SSL/TLS)
Always use SSL/TLS for client-to-server connections and force encryption on the server.
```sql
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'ForceEncryption', 1; RECONFIGURE;
```
Configure your connection strings to include `Encrypt=True`.

## 2. Transparent Data Encryption (TDE)
Encrypts the database files (`.mdf`, `.ndf`, `.ldf`) and backups. Operates at the page level.
- **Best Use Case:** Full database encryption for compliance (e.g., GDPR, HIPAA).
- **Setup:** Create a master key, certificate, and DEK. Enable encryption on the database.

## 3. Always Encrypted
End-to-end encryption. Data is encrypted/decrypted at the client level, meaning SQL Server never sees the plaintext.
- **Best Use Case:** Highly sensitive columns (e.g., SSN, Credit Card numbers).
- **Setup:** Create a column master key (CMK) and column encryption key (CEK). Configure the application with the appropriate driver.

## 4. Dynamic Data Masking (DDM)
Obfuscates data for non-privileged users without changing the underlying data.
- **Best Use Case:** Protecting data from being viewed by unauthorized users (e.g., masking credit card numbers in the application UI).
- **Setup:** Add a mask to a column using the `MASKED WITH` clause.
```sql
ALTER TABLE Sales.Customers
ALTER COLUMN CreditCardNumber MASKED WITH (FUNCTION = 'partial(0,"XXXX-XXXX-XXXX-",4)');
```

## 5. Column-Level Encryption (Cell-Level Encryption)
Encrypts individual columns using symmetric or asymmetric keys.
- **Best Use Case:** Granular encryption for specific data elements within a row.
- **Setup:** Create a symmetric key and certificate. Use `EncryptByKey` and `DecryptByKey`.

## Best Practices
- **Certificate Management:** Regularly back up your certificates and master keys to a secure location.
- **Key Rotation:** Implement a regular key rotation policy for certificates and column encryption keys.
- **Performance:** Be aware of the performance impact of encryption, especially for Always Encrypted and Column-Level Encryption.

## References
- [Microsoft Documentation: SQL Server Security - Encryption](https://learn.microsoft.com/en-us/sql/relational-databases/security/encryption/sql-server-encryption)
- [Microsoft Documentation: Transparent Data Encryption (TDE)](https://learn.microsoft.com/en-us/sql/relational-databases/security/encryption/transparent-data-encryption)
- [Microsoft Documentation: Always Encrypted](https://learn.microsoft.com/en-us/sql/relational-databases/security/encryption/always-encrypted-database-engine)
