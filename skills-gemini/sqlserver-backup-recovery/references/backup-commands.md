# Backup Commands Guide

A comprehensive list of common SQL Server backup commands and their options.

## 1. Full Database Backup
```sql
BACKUP DATABASE [MyDatabase]
TO DISK = 'C:\Backups\MyDatabase_Full.bak'
WITH CHECKSUM, COMPRESSION, STATS = 10, INIT, FORMAT;
```
- **CHECKSUM:** Verifies page checksums during the backup.
- **COMPRESSION:** Reduces the backup file size and I/O.
- **STATS = 10:** Reports the progress every 10%.
- **INIT:** Overwrites the backup file (media).
- **FORMAT:** Creates a new media header.

## 2. Differential Database Backup
```sql
BACKUP DATABASE [MyDatabase]
TO DISK = 'C:\Backups\MyDatabase_Diff.bak'
WITH DIFFERENTIAL, CHECKSUM, COMPRESSION, STATS = 10;
```
- **DIFFERENTIAL:** Backs up only the changes since the last full backup.

## 3. Transaction Log Backup
```sql
BACKUP LOG [MyDatabase]
TO DISK = 'C:\Backups\MyDatabase_Log.trn'
WITH CHECKSUM, COMPRESSION, STATS = 10;
```
- **LOG:** Backs up the transaction log. Essential for Full recovery model.

## 4. Copy-Only Backup
```sql
BACKUP DATABASE [MyDatabase]
TO DISK = 'C:\Backups\MyDatabase_CopyOnly.bak'
WITH COPY_ONLY, CHECKSUM, COMPRESSION;
```
- **COPY_ONLY:** Doesn't affect the LSN (Log Sequence Number) chain or the differential base. Useful for one-off backups.

## 5. Backup to URL (Azure Blob Storage)
```sql
BACKUP DATABASE [MyDatabase]
TO URL = 'https://mystorage.blob.core.windows.net/backups/MyDatabase_Full.bak'
WITH CREDENTIAL = 'MyAzureCredential', COMPRESSION, STATS = 10;
```
- **URL:** Direct backup to Azure storage using a SAS token.

## 6. Striped Backup
```sql
BACKUP DATABASE [MyDatabase]
TO DISK = 'C:\Backups\MyDatabase_Part1.bak',
   DISK = 'D:\Backups\MyDatabase_Part2.bak'
WITH CHECKSUM, COMPRESSION, STATS = 10;
```
- **Striping:** Splits the backup into multiple files for better I/O performance on large databases.

## References
- [Microsoft Documentation: BACKUP (Transact-SQL)](https://learn.microsoft.com/en-us/sql/t-sql/statements/backup-transact-sql)
- [Ola Hallengren's SQL Server Backup Script](https://ola.hallengren.com/sql-server-backup.html)
