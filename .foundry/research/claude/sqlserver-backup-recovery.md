# SQL Server Backup and Recovery Strategies

## Overview

SQL Server backup and recovery is governed by the recovery model of each database and the RTO/RPO requirements of the business. The three recovery models are FULL, BULK_LOGGED, and SIMPLE. A comprehensive strategy combines full, differential, and transaction log backups, tested regularly with restore verification.

---

## Recovery Models

### FULL Recovery Model
- All transactions logged; point-in-time recovery is possible.
- Log space is only reclaimed after a log backup (not checkpoint).
- Required for: log shipping, Always On Availability Groups, database mirroring.
- Use for: all production databases with RPO requirements measured in minutes or seconds.

### BULK_LOGGED Recovery Model
- Minimally logs bulk operations (BULK INSERT, SELECT INTO, index rebuilds).
- Log backups cannot restore to a point in time during bulk operations.
- Use for: ETL periods with large bulk loads, then switch back to FULL.
- Risk: log backup after bulk operation captures full extent of bulk changes.

### SIMPLE Recovery Model
- Log is automatically truncated at each checkpoint.
- No log backups possible. Recovery only to last full or differential backup.
- Use for: development, QA, tempdb (always SIMPLE), databases with no RPO requirement.

```sql
-- Check recovery model
SELECT name, recovery_model_desc, log_reuse_wait_desc FROM sys.databases;

-- Change recovery model
ALTER DATABASE [YourDB] SET RECOVERY FULL;
ALTER DATABASE [YourDB] SET RECOVERY BULK_LOGGED;
ALTER DATABASE [YourDB] SET RECOVERY SIMPLE;
```

---

## Backup Types

### Full Backup
- Copies all allocated pages in the database at the time of backup.
- The baseline for all other backup types.
- Can be performed online (database remains accessible).
- Duration/size scales with database size.

### Differential Backup
- Contains all pages modified since the last FULL backup (not since last differential).
- Grows as more changes accumulate since the last full.
- Faster to take than full; faster to restore than many log backups.
- Restore sequence: Full → latest Differential → Log backups after differential.

### Transaction Log Backup
- Contains all transactions since the last log backup.
- Enables point-in-time recovery.
- Required in FULL or BULK_LOGGED recovery models to prevent log growth.
- Restore sequence: Full → [Differential] → all Log backups in order.

### Copy-Only Backup
- Takes a full or log backup outside the normal backup chain.
- Does NOT affect the differential base or log chain.
- Use for: ad hoc backups before major changes, seeding Always On replicas.

```sql
BACKUP DATABASE [YourDB] TO DISK = 'D:\Backups\YourDB_COPY.bak'
WITH COPY_ONLY, COMPRESSION, CHECKSUM, STATS = 10;
```

### File/Filegroup Backup
- Backs up individual files or filegroups.
- Useful for very large databases (VLDBs) where full backup is impractical.
- Requires log backups to maintain consistency across files.

---

## Backup Commands

### Full Backup
```sql
BACKUP DATABASE [YourDB]
TO DISK = N'D:\Backups\YourDB_Full_20240115.bak'
WITH
    COMPRESSION,           -- Reduce backup size (CPU overhead)
    CHECKSUM,              -- Verify page checksums during backup
    STATS = 10,            -- Progress report every 10%
    NAME = N'YourDB Full Backup',
    DESCRIPTION = N'Weekly full backup';
```

### Differential Backup
```sql
BACKUP DATABASE [YourDB]
TO DISK = N'D:\Backups\YourDB_Diff_20240115_1800.bak'
WITH DIFFERENTIAL, COMPRESSION, CHECKSUM, STATS = 10;
```

### Transaction Log Backup
```sql
BACKUP LOG [YourDB]
TO DISK = N'D:\Backups\YourDB_Log_20240115_1800.trn'
WITH COMPRESSION, CHECKSUM, STATS = 10;
```

### Striped Backup (multiple files, faster for large DBs)
```sql
BACKUP DATABASE [YourDB]
TO
    DISK = N'D:\Backups\YourDB_Full_1.bak',
    DISK = N'E:\Backups\YourDB_Full_2.bak',
    DISK = N'F:\Backups\YourDB_Full_3.bak'
WITH COMPRESSION, CHECKSUM, STATS = 10;
```

### Backup to URL (Azure Blob Storage)
```sql
-- Requires credential first
CREATE CREDENTIAL [https://storageaccount.blob.core.windows.net/container]
WITH IDENTITY = 'SHARED ACCESS SIGNATURE',
SECRET = 'sv=2020-08-04&ss=b&srt=...';

BACKUP DATABASE [YourDB]
TO URL = N'https://storageaccount.blob.core.windows.net/container/YourDB_Full.bak'
WITH COMPRESSION, CHECKSUM, STATS = 10;
```

---

## Restore Commands

### Full Restore (replace existing database)
```sql
-- Step 1: Tail-log backup (capture transactions not yet backed up)
BACKUP LOG [YourDB]
TO DISK = N'D:\Backups\YourDB_TailLog.trn'
WITH NO_TRUNCATE, NORECOVERY, CHECKSUM;

-- Step 2: Restore full backup (NORECOVERY = leave DB in restoring state)
RESTORE DATABASE [YourDB]
FROM DISK = N'D:\Backups\YourDB_Full_20240115.bak'
WITH NORECOVERY, REPLACE, STATS = 10;

-- Step 3: Restore differential (if exists, use most recent)
RESTORE DATABASE [YourDB]
FROM DISK = N'D:\Backups\YourDB_Diff_20240115_1800.bak'
WITH NORECOVERY, STATS = 10;

-- Step 4: Restore all log backups in chronological order
RESTORE LOG [YourDB]
FROM DISK = N'D:\Backups\YourDB_Log_20240115_1900.trn'
WITH NORECOVERY, STATS = 10;

RESTORE LOG [YourDB]
FROM DISK = N'D:\Backups\YourDB_Log_20240115_2000.trn'
WITH NORECOVERY, STATS = 10;

-- Step 5: Bring database online
RESTORE DATABASE [YourDB] WITH RECOVERY;
```

### Point-in-Time Restore
```sql
-- Restore to a specific point in time (must restore logs in NORECOVERY until desired time)
RESTORE LOG [YourDB]
FROM DISK = N'D:\Backups\YourDB_Log_20240115_2000.trn'
WITH RECOVERY, STOPAT = N'2024-01-15 19:45:00';
```

### Restore to Different Server/Database Name
```sql
RESTORE DATABASE [YourDB_Restored]
FROM DISK = N'D:\Backups\YourDB_Full_20240115.bak'
WITH
    MOVE N'YourDB_Data' TO N'D:\Data\YourDB_Restored.mdf',
    MOVE N'YourDB_Log'  TO N'D:\Log\YourDB_Restored_log.ldf',
    RECOVERY, REPLACE, STATS = 10;
```

### Verify Backup Without Restoring
```sql
RESTORE VERIFYONLY
FROM DISK = N'D:\Backups\YourDB_Full_20240115.bak'
WITH CHECKSUM;
```

### Get Backup Header Information
```sql
-- List contents of a backup file
RESTORE HEADERONLY FROM DISK = N'D:\Backups\YourDB_Full_20240115.bak';

-- List files in backup
RESTORE FILELISTONLY FROM DISK = N'D:\Backups\YourDB_Full_20240115.bak';

-- Get label information
RESTORE LABELONLY FROM DISK = N'D:\Backups\YourDB_Full_20240115.bak';
```

---

## Backup History Queries

### Recent Backup History
```sql
SELECT TOP 50
    bs.database_name,
    bs.type AS backup_type,
    CASE bs.type
        WHEN 'D' THEN 'Full'
        WHEN 'I' THEN 'Differential'
        WHEN 'L' THEN 'Log'
        WHEN 'F' THEN 'File'
    END AS backup_type_desc,
    bs.backup_start_date,
    bs.backup_finish_date,
    DATEDIFF(MINUTE, bs.backup_start_date, bs.backup_finish_date) AS duration_min,
    bs.backup_size / 1024.0 / 1024.0 / 1024.0 AS size_gb,
    bs.compressed_backup_size / 1024.0 / 1024.0 / 1024.0 AS compressed_gb,
    CAST(100.0 * bs.compressed_backup_size / NULLIF(bs.backup_size, 0) AS DECIMAL(5,2)) AS compression_pct,
    bmf.physical_device_name AS backup_file
FROM msdb.dbo.backupset bs
JOIN msdb.dbo.backupmediafamily bmf ON bs.media_set_id = bmf.media_set_id
ORDER BY bs.backup_start_date DESC;
```

### Databases Without Recent Full Backup
```sql
SELECT
    d.name AS database_name,
    d.recovery_model_desc,
    MAX(bs.backup_finish_date) AS last_full_backup,
    DATEDIFF(HOUR, MAX(bs.backup_finish_date), GETDATE()) AS hours_since_backup
FROM sys.databases d
LEFT JOIN msdb.dbo.backupset bs
    ON d.name = bs.database_name AND bs.type = 'D'
WHERE d.database_id > 4  -- Exclude system databases
GROUP BY d.name, d.recovery_model_desc
HAVING MAX(bs.backup_finish_date) IS NULL
    OR MAX(bs.backup_finish_date) < DATEADD(HOUR, -25, GETDATE())
ORDER BY hours_since_backup DESC;
```

### Databases Without Recent Log Backup (FULL recovery)
```sql
SELECT
    d.name AS database_name,
    MAX(bs.backup_finish_date) AS last_log_backup,
    DATEDIFF(MINUTE, MAX(bs.backup_finish_date), GETDATE()) AS minutes_since_log_backup
FROM sys.databases d
LEFT JOIN msdb.dbo.backupset bs
    ON d.name = bs.database_name AND bs.type = 'L'
WHERE d.database_id > 4
  AND d.recovery_model_desc = 'FULL'
  AND d.state = 0  -- Online databases
GROUP BY d.name
HAVING MAX(bs.backup_finish_date) IS NULL
    OR MAX(bs.backup_finish_date) < DATEADD(MINUTE, -60, GETDATE())
ORDER BY minutes_since_log_backup DESC;
```

### Restore Chain Validation
```sql
-- Check that log chain is unbroken for a database
SELECT
    database_name,
    type,
    backup_start_date,
    backup_finish_date,
    first_lsn,
    last_lsn,
    checkpoint_lsn,
    database_backup_lsn
FROM msdb.dbo.backupset
WHERE database_name = 'YourDB'
  AND backup_start_date >= DATEADD(DAY, -7, GETDATE())
ORDER BY backup_start_date;
```

---

## Backup Strategy Templates

### Standard OLTP Strategy (RPO = 15 min, RTO = 1 hour)
```
Sunday 11 PM:    Full backup
Mon-Sat 11 PM:   Differential backup
Every 15 min:    Transaction log backup
Retention:       Full = 4 weeks, Diff = 2 weeks, Log = 1 week
```

### High Availability Strategy (RPO = 0, RTO = seconds)
```
Always On AG (synchronous):  RPO = 0 (zero data loss)
Frequent log backups:        Every 5 minutes on secondary
Automated failover:          Configured with Windows Server Failover Cluster
DR site (async):             15-minute lag acceptable
```

### Dev/Test Strategy
```
Weekly full backup only (SIMPLE recovery model)
No log backups needed
Shorter retention (7 days)
```

---

## SQL Server Agent Backup Jobs

```sql
-- Create a maintenance plan-style backup job using T-SQL
USE msdb;
GO

EXEC sp_add_job @job_name = N'DBA - Full Backup All User DBs';

EXEC sp_add_jobstep
    @job_name = N'DBA - Full Backup All User DBs',
    @step_name = N'Run Full Backup',
    @command = N'
DECLARE @db NVARCHAR(128), @path NVARCHAR(500), @date NVARCHAR(20);
SET @date = FORMAT(GETDATE(), ''yyyyMMdd_HHmm'');
DECLARE db_cursor CURSOR FOR
    SELECT name FROM sys.databases
    WHERE database_id > 4 AND state = 0;
OPEN db_cursor;
FETCH NEXT FROM db_cursor INTO @db;
WHILE @@FETCH_STATUS = 0
BEGIN
    SET @path = N''D:\Backups\'' + @db + N''_Full_'' + @date + N''.bak'';
    BACKUP DATABASE @db TO DISK = @path
    WITH COMPRESSION, CHECKSUM, STATS = 10;
    FETCH NEXT FROM db_cursor INTO @db;
END;
CLOSE db_cursor;
DEALLOCATE db_cursor;',
    @subsystem = N'TSQL';

EXEC sp_add_schedule
    @schedule_name = N'Weekly Sunday 11PM',
    @freq_type = 8,           -- Weekly
    @freq_interval = 1,       -- Sunday
    @active_start_time = 230000;  -- 11 PM

EXEC sp_attach_schedule
    @job_name = N'DBA - Full Backup All User DBs',
    @schedule_name = N'Weekly Sunday 11PM';

EXEC sp_add_jobserver @job_name = N'DBA - Full Backup All User DBs';
```

---

## Backup Compression and Encryption

### Compression
```sql
-- Enable instance-level backup compression
EXEC sp_configure 'backup compression default', 1;
RECONFIGURE;

-- Per-backup override
BACKUP DATABASE [YourDB] TO DISK = '...' WITH NO_COMPRESSION;  -- Override to disable
BACKUP DATABASE [YourDB] TO DISK = '...' WITH COMPRESSION;     -- Enable for this backup
```

### Backup Encryption (SQL 2014+)
```sql
-- Step 1: Create master key in master database
USE master;
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'StrongPassword123!';

-- Step 2: Create certificate for backup encryption
CREATE CERTIFICATE BackupEncryptCert
WITH SUBJECT = 'Database Backup Encryption Certificate';

-- Step 3: Back up the certificate (CRITICAL - store securely off-server)
BACKUP CERTIFICATE BackupEncryptCert
TO FILE = 'D:\Certs\BackupEncryptCert.cer'
WITH PRIVATE KEY (
    FILE = 'D:\Certs\BackupEncryptCert.pvk',
    ENCRYPTION BY PASSWORD = 'CertPassword456!'
);

-- Step 4: Encrypted backup
BACKUP DATABASE [YourDB]
TO DISK = N'D:\Backups\YourDB_Encrypted.bak'
WITH COMPRESSION, ENCRYPTION (ALGORITHM = AES_256, SERVER CERTIFICATE = BackupEncryptCert), STATS = 10;
```

---

## Common Issues and Solutions

### Log File Grows Uncontrollably
- Cause: FULL recovery model with no log backups being taken.
- Check: `log_reuse_wait_desc` in `sys.databases`. Value 'LOG_BACKUP' = log backup needed.
- Fix: Take a log backup immediately, then schedule regular log backups.

### Backup Job Fails with "Media family error"
- Cause: Backup file exists from a previous backup set with INIT not specified.
- Fix: Use `WITH INIT` to overwrite, or delete old file first. Use `WITH NOINIT` to append.

### Restore Fails: "The backup set holds a backup of a database other than the existing database"
- Fix: Use `WITH REPLACE` to override the check:
  `RESTORE DATABASE [YourDB] FROM DISK = '...' WITH REPLACE, RECOVERY;`

### Backup to Network Share is Slow
- Use a UNC path: `\\server\share\file.bak`
- Ensure SQL Server service account has write access.
- Prefer local disk or dedicated backup infrastructure.

### LSN Chain Broken (cannot restore logs)
- Cause: A full or copy-only backup was taken that was not known to the restore chain.
- Fix: Restore from the most recent full backup that starts a new chain.

---

## Key Monitoring Metrics

| Metric | Tool | Threshold |
|--------|------|-----------|
| Last full backup age | msdb.dbo.backupset | Alert if > 24 hours |
| Last log backup age (FULL) | msdb.dbo.backupset | Alert if > 60 min |
| Backup duration trend | msdb.dbo.backupset | Alert on > 20% increase |
| Log space used % | sys.dm_db_log_space_usage | Alert if > 80% |
| VLF count | sys.dm_db_log_info | Alert if > 1000 |
| Backup file verify (CHECKSUM) | RESTORE VERIFYONLY | Failed = critical |

---

## Ola Hallengren Backup Solution (Industry Standard)

Ola Hallengren's SQL Server Maintenance Solution is the de-facto standard for automated backups:
- `DatabaseBackup` stored procedure handles full/differential/log backups with compression, checksums, cleanup, and logging.
- Supports backup to disk, network, Azure Blob, and NUL (for testing).
- Source: https://ola.hallengren.com/sql-server-backup.html

```sql
-- Example call (after installing Ola's solution)
EXECUTE dbo.DatabaseBackup
    @Databases = 'USER_DATABASES',
    @Directory = N'D:\Backups',
    @BackupType = 'FULL',
    @Verify = 'Y',
    @Compress = 'Y',
    @CheckSum = 'Y',
    @CleanupTime = 168;  -- Delete files older than 168 hours (1 week)
```
