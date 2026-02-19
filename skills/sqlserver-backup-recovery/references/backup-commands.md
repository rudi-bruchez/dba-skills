# SQL Server Backup and Restore Commands — Complete Reference

---

## Backup Commands

### Full Database Backup
```sql
-- Minimal
BACKUP DATABASE [YourDB]
TO DISK = N'D:\Backups\YourDB_Full.bak'
WITH COMPRESSION, CHECKSUM, STATS = 10;

-- Full options
BACKUP DATABASE [YourDB]
TO DISK = N'D:\Backups\YourDB_Full_20240115_2300.bak'
WITH
    COMPRESSION,                          -- Compress backup file
    CHECKSUM,                             -- Verify page checksums; detect corruption
    STATS = 10,                           -- Progress every 10%
    INIT,                                 -- Overwrite existing backup file
    NAME = N'YourDB Weekly Full Backup',  -- Backup set name (visible in HEADERONLY)
    DESCRIPTION = N'Weekly full backup taken Sunday 11 PM',
    EXPIREDATE = '2024-02-15',            -- Backup expires after this date
    RETAINDAYS = 30;                      -- Alternative: retain for N days
```

### Differential Backup
```sql
BACKUP DATABASE [YourDB]
TO DISK = N'D:\Backups\YourDB_Diff_20240115_1800.bak'
WITH DIFFERENTIAL, COMPRESSION, CHECKSUM, STATS = 10;
-- Captures all pages changed since the last FULL backup (not since last differential)
-- Restore order: Full → latest Differential → Log backups after differential
```

### Transaction Log Backup
```sql
BACKUP LOG [YourDB]
TO DISK = N'D:\Backups\YourDB_Log_20240115_1815.trn'
WITH COMPRESSION, CHECKSUM, STATS = 10;
-- Only valid in FULL or BULK_LOGGED recovery model
-- Truncates inactive portion of the log after backup completes
```

### Tail-Log Backup (before restore)
```sql
-- Captures transactions not yet backed up. Run before restoring.
BACKUP LOG [YourDB]
TO DISK = N'D:\Backups\YourDB_TailLog_20240115.trn'
WITH NO_TRUNCATE,     -- Allow backup even if database is damaged/inaccessible
     NORECOVERY,      -- Leave database in RESTORING state for subsequent restore
     CHECKSUM;
```

### Copy-Only Backup (does not affect backup chain)
```sql
BACKUP DATABASE [YourDB]
TO DISK = N'D:\Backups\YourDB_CopyOnly.bak'
WITH COPY_ONLY, COMPRESSION, CHECKSUM, STATS = 10;
-- Does not reset the differential base
-- Use for: ad hoc backups, seeding Always On replicas, pre-deployment backups
```

### Striped Backup (multiple files for faster throughput)
```sql
BACKUP DATABASE [YourDB]
TO DISK = N'D:\Backups\YourDB_Full_1.bak',
   DISK = N'E:\Backups\YourDB_Full_2.bak',
   DISK = N'F:\Backups\YourDB_Full_3.bak'
WITH COMPRESSION, CHECKSUM, STATS = 10;
-- All files are required for restore; keep them together
```

### Backup to Azure Blob Storage
```sql
-- Create a credential with a SAS token first
CREATE CREDENTIAL [https://storageacct.blob.core.windows.net/backups]
WITH IDENTITY = 'SHARED ACCESS SIGNATURE',
SECRET = 'sv=2020-08-04&ss=b&...';  -- SAS token (without leading ?)

BACKUP DATABASE [YourDB]
TO URL = N'https://storageacct.blob.core.windows.net/backups/YourDB_Full.bak'
WITH COMPRESSION, CHECKSUM, STATS = 10;
```

### Encrypted Backup (SQL 2014+)
```sql
-- Prerequisites: master key + certificate (see encryption-guide.md)
BACKUP DATABASE [YourDB]
TO DISK = N'D:\Backups\YourDB_Encrypted.bak'
WITH
    COMPRESSION,
    ENCRYPTION (ALGORITHM = AES_256, SERVER CERTIFICATE = BackupEncryptCert),
    STATS = 10;
```

---

## Backup Verification Commands

```sql
-- Verify backup file is readable and internally consistent
RESTORE VERIFYONLY
FROM DISK = N'D:\Backups\YourDB_Full.bak'
WITH CHECKSUM;
-- Returns "The backup set on file 1 is valid" if successful

-- Read all backup sets in the file (metadata)
RESTORE HEADERONLY FROM DISK = N'D:\Backups\YourDB_Full.bak';
-- Shows: DatabaseName, BackupType, BackupStartDate, FirstLSN, LastLSN, BackupSizeMB

-- Read logical and physical file names (needed for MOVE during restore)
RESTORE FILELISTONLY FROM DISK = N'D:\Backups\YourDB_Full.bak';
-- Shows: LogicalName, PhysicalName, Type (D=data, L=log)

-- Read media label
RESTORE LABELONLY FROM DISK = N'D:\Backups\YourDB_Full.bak';
```

---

## Restore Commands

### Full Restore (overwrite existing database)
```sql
-- Step 1: Tail-log backup (if database is accessible)
BACKUP LOG [YourDB] TO DISK = N'D:\Backups\YourDB_TailLog.trn'
WITH NO_TRUNCATE, NORECOVERY, CHECKSUM;

-- Step 2: Restore full backup
RESTORE DATABASE [YourDB]
FROM DISK = N'D:\Backups\YourDB_Full_20240115.bak'
WITH NORECOVERY, REPLACE, STATS = 10;

-- Step 3: Restore differential (most recent)
RESTORE DATABASE [YourDB]
FROM DISK = N'D:\Backups\YourDB_Diff_20240115_1800.bak'
WITH NORECOVERY, STATS = 10;

-- Step 4: Restore log backups in order
RESTORE LOG [YourDB] FROM DISK = N'D:\Backups\YourDB_Log_20240115_1900.trn'
WITH NORECOVERY;
RESTORE LOG [YourDB] FROM DISK = N'D:\Backups\YourDB_Log_20240115_2000.trn'
WITH NORECOVERY;

-- Step 5: Bring online
RESTORE DATABASE [YourDB] WITH RECOVERY;
```

### Point-in-Time Restore
```sql
-- On the last log backup in the chain (the one spanning the target time):
RESTORE LOG [YourDB]
FROM DISK = N'D:\Backups\YourDB_Log_20240115_2100.trn'
WITH RECOVERY, STOPAT = N'2024-01-15 20:47:00';
-- STOPAT must be within the time range covered by this log backup
```

### Restore to Different Name and Path
```sql
-- Get logical names first:
RESTORE FILELISTONLY FROM DISK = N'D:\Backups\YourDB_Full.bak';

-- Restore to new name and path:
RESTORE DATABASE [YourDB_Restored]
FROM DISK = N'D:\Backups\YourDB_Full.bak'
WITH
    MOVE N'YourDB'     TO N'E:\Data\YourDB_Restored.mdf',
    MOVE N'YourDB_log' TO N'E:\Log\YourDB_Restored_log.ldf',
    RECOVERY, REPLACE, STATS = 10;
```

### Restore Specific File Number from Striped Backup
```sql
-- HEADERONLY shows FILE number if multiple backup sets in one file
RESTORE DATABASE [YourDB]
FROM DISK = N'D:\Backups\YourDB_Backups.bak'
WITH FILE = 2,  -- Restore the second backup set
     NORECOVERY, REPLACE, STATS = 10;
```

### Restore Single Page (suspect page)
```sql
-- For a single corrupt page (get page ID from suspect_pages system table)
RESTORE DATABASE [YourDB] PAGE = '1:51423'
FROM DISK = N'D:\Backups\YourDB_Full.bak'
WITH NORECOVERY;
-- Then restore log backups to bring the page current
RESTORE LOG [YourDB] FROM DISK = N'D:\Backups\YourDB_Log.trn' WITH RECOVERY;
```

---

## Instance-Level Configuration

```sql
-- Enable backup compression as default
EXEC sp_configure 'backup compression default', 1;
RECONFIGURE;

-- Check current backup settings
SELECT name, value_in_use
FROM sys.configurations
WHERE name LIKE '%backup%';

-- Check suspect pages (pages flagged by checksum failures during backup/restore)
SELECT * FROM msdb.dbo.suspect_pages WHERE event_type > 0;
-- event_type: 1=checksum error, 2=torn page, 3=other I/O error
```

---

## Backup Chain Validation

```sql
-- Check LSN chain for restore validity
SELECT
    database_name,
    CASE type WHEN 'D' THEN 'Full' WHEN 'I' THEN 'Diff' WHEN 'L' THEN 'Log' END AS type,
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
-- For a valid restore chain: each log backup's first_lsn must <= previous last_lsn
```
