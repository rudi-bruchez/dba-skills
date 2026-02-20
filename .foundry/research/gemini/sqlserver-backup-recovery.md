# SQL Server Backup & Recovery Research

## Overview
SQL Server backup and recovery are the primary defense against data loss. A robust strategy ensures that data is durable and restorable within the required Recovery Point Objective (RPO) and Recovery Time Objective (RTO).

## Recovery Models
- **Simple Recovery Model:** No transaction log backups. Least disk usage, but can only restore to the last full/differential backup.
- **Full Recovery Model:** All transactions are logged. Required for point-in-time recovery. Transaction logs must be backed up to prevent them from filling the disk.
- **Bulk-Logged Recovery Model:** Minimizes logging for bulk operations (e.g., `BULK INSERT`, `SELECT INTO`). Does not support point-in-time recovery for bulk transactions.

## Backup Types
- **Full Backup:** Entire database. The baseline for all other backups.
- **Differential Backup:** Changes since the last full backup. Smaller and faster than full backups.
- **Transaction Log Backup:** Changes since the last log backup. Enables point-in-time recovery.
- **File / Filegroup Backup:** For large databases where backing up the entire database is not practical.
- **Copy-Only Backup:** A full or log backup that doesn't break the LSN (Log Sequence Number) chain. Used for one-off backups.

## Backup Verification & Integrity
- **`WITH CHECKSUM`:** Instructs SQL Server to verify data page checksums during backup.
- **`RESTORE VERIFYONLY`:** Checks the backup file's headers and readability without actually restoring the data.
- **`DBCC CHECKDB`:** Essential for verifying the physical and logical integrity of the database before taking a backup.

## Disaster Recovery Strategies
- **The 3-2-1-1-0 Rule (Modern):** 
    - **3** copies of data.
    - **2** different media types.
    - **1** copy offsite.
    - **1** copy offline (air-gapped/immutable).
    - **0** errors (verified through testing).
- **RPO (Recovery Point Objective):** Maximum tolerable data loss (e.g., 15 minutes of data).
- **RTO (Recovery Time Objective):** Maximum tolerable downtime (e.g., 4 hours to recover).

## Best Practices (2024-2025)
- **Backup Compression:** Use native SQL Server backup compression (or Zstandard/ZSTD in SQL 2025) to save disk space and I/O.
- **Backup Encryption:** Protect sensitive data in backup files using certificates or asymmetric keys.
- **Cloud-Integrated Backups:** Direct backup to Azure Blob Storage (using URL) for offsite durability.
- **Regular Restore Testing:** Automate the process of restoring backups to a dev/test environment to ensure they are valid.
- **Managed Backup:** Use Azure SQL Managed Instance or AWS RDS for managed backup/recovery if moving to the cloud.

## Key T-SQL Commands
- `BACKUP DATABASE [DBName] TO DISK = 'C:\Backups\DBName.bak' WITH CHECKSUM, COMPRESSION, STATS = 10;`
- `RESTORE DATABASE [DBName] FROM DISK = 'C:\Backups\DBName.bak' WITH NORECOVERY, STATS = 10;`
- `BACKUP LOG [DBName] TO DISK = 'C:\Backups\DBName_Log.trn' WITH CHECKSUM, COMPRESSION, STATS = 10;`

## References
- [Microsoft Documentation: Backup and Restore](https://learn.microsoft.com/en-us/sql/relational-databases/backup-restore/back-up-and-restore-of-sql-server-databases)
- [Ola Hallengren's SQL Server Maintenance Solution](https://ola.hallengren.com/sql-server-backup.html) (De facto standard for T-SQL backups)
- [Brent Ozar's Guide to SQL Server Backups](https://www.brentozar.com/archive/2014/06/sql-server-backup-strategy/)
