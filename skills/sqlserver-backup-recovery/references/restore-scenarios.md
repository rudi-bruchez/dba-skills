# SQL Server Restore Scenarios — Step-by-Step Procedures

---

## Scenario 1: Full Database Restore (Replace Existing)

Use when: corrupted database, accidental data deletion affecting the entire database, disaster recovery.

```sql
-- Step 1: Identify available backups
SELECT database_name,
       CASE type WHEN 'D' THEN 'Full' WHEN 'I' THEN 'Diff' WHEN 'L' THEN 'Log' END AS type,
       backup_start_date, backup_finish_date,
       physical_device_name
FROM msdb.dbo.backupset bs
JOIN msdb.dbo.backupmediafamily bmf ON bs.media_set_id = bmf.media_set_id
WHERE database_name = 'YourDB'
  AND backup_start_date >= DATEADD(DAY, -7, GETDATE())
ORDER BY backup_start_date;

-- Step 2: Tail-log backup (if database is still accessible — preserves recent transactions)
BACKUP LOG [YourDB]
TO DISK = N'D:\Backups\YourDB_TailLog_EMERGENCY.trn'
WITH NO_TRUNCATE, NORECOVERY, CHECKSUM;

-- Step 3: Take any active users offline
-- Option A: Set single user (kicks active connections)
ALTER DATABASE [YourDB] SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
-- Option B: Kill individual sessions
-- SELECT session_id FROM sys.dm_exec_sessions WHERE database_id = DB_ID('YourDB')
-- KILL <session_id>;

-- Step 4: Restore the full backup
RESTORE DATABASE [YourDB]
FROM DISK = N'D:\Backups\YourDB_Full_20240115.bak'
WITH NORECOVERY, REPLACE, STATS = 10;

-- Step 5: Restore the most recent differential backup (if available)
RESTORE DATABASE [YourDB]
FROM DISK = N'D:\Backups\YourDB_Diff_20240115_2300.bak'
WITH NORECOVERY, STATS = 10;

-- Step 6: Restore all log backups after the differential, in chronological order
RESTORE LOG [YourDB] FROM DISK = N'D:\Backups\YourDB_Log_20240116_0000.trn' WITH NORECOVERY;
RESTORE LOG [YourDB] FROM DISK = N'D:\Backups\YourDB_Log_20240116_0015.trn' WITH NORECOVERY;
RESTORE LOG [YourDB] FROM DISK = N'D:\Backups\YourDB_TailLog_EMERGENCY.trn' WITH NORECOVERY;

-- Step 7: Bring database online
RESTORE DATABASE [YourDB] WITH RECOVERY;

-- Step 8: Verify and re-enable multi-user access
SELECT state_desc FROM sys.databases WHERE name = 'YourDB';  -- Should be ONLINE
ALTER DATABASE [YourDB] SET MULTI_USER;
```

---

## Scenario 2: Point-in-Time Recovery (Specific Time)

Use when: accidental `DELETE` or `UPDATE` without `WHERE`, data corruption at a known time.

**Known problem time: 2024-01-15 14:37:00**

```sql
-- Step 1: Find the full backup that precedes the problem
SELECT backup_start_date, backup_finish_date, physical_device_name
FROM msdb.dbo.backupset bs
JOIN msdb.dbo.backupmediafamily bmf ON bs.media_set_id = bmf.media_set_id
WHERE database_name = 'YourDB' AND type = 'D'
  AND backup_start_date <= '2024-01-15 14:37:00'
ORDER BY backup_start_date DESC;
-- Use most recent: D:\Backups\YourDB_Full_20240114_2300.bak (Sunday night)

-- Step 2: Find the most recent differential before the problem time
SELECT backup_start_date, backup_finish_date, physical_device_name
FROM msdb.dbo.backupset bs
JOIN msdb.dbo.backupmediafamily bmf ON bs.media_set_id = bmf.media_set_id
WHERE database_name = 'YourDB' AND type = 'I'
  AND backup_start_date BETWEEN '2024-01-14 23:00' AND '2024-01-15 14:37:00'
ORDER BY backup_start_date DESC;
-- Most recent: D:\Backups\YourDB_Diff_20240115_1200.bak (noon differential)

-- Step 3: List all log backups after the differential and before the problem
SELECT backup_start_date, backup_finish_date, physical_device_name
FROM msdb.dbo.backupset bs
JOIN msdb.dbo.backupmediafamily bmf ON bs.media_set_id = bmf.media_set_id
WHERE database_name = 'YourDB' AND type = 'L'
  AND backup_start_date BETWEEN '2024-01-15 12:00' AND '2024-01-15 14:37:00'
ORDER BY backup_start_date;

-- Step 4: Restore to a different name (to validate before replacing production)
RESTORE DATABASE [YourDB_PITR]
FROM DISK = N'D:\Backups\YourDB_Full_20240114_2300.bak'
WITH NORECOVERY, REPLACE,
     MOVE N'YourDB'     TO N'D:\Data\YourDB_PITR.mdf',
     MOVE N'YourDB_log' TO N'D:\Log\YourDB_PITR_log.ldf',
     STATS = 10;

RESTORE DATABASE [YourDB_PITR]
FROM DISK = N'D:\Backups\YourDB_Diff_20240115_1200.bak'
WITH NORECOVERY, STATS = 10;

RESTORE LOG [YourDB_PITR] FROM DISK = N'D:\Backups\YourDB_Log_20240115_1215.trn' WITH NORECOVERY;
RESTORE LOG [YourDB_PITR] FROM DISK = N'D:\Backups\YourDB_Log_20240115_1230.trn' WITH NORECOVERY;
RESTORE LOG [YourDB_PITR] FROM DISK = N'D:\Backups\YourDB_Log_20240115_1245.trn' WITH NORECOVERY;
-- ... (continue through logs)

-- Step 5: Apply the log that spans the target time with STOPAT
RESTORE LOG [YourDB_PITR]
FROM DISK = N'D:\Backups\YourDB_Log_20240115_1430.trn'
WITH RECOVERY, STOPAT = N'2024-01-15 14:36:59';
-- Stop 1 second before the mistake

-- Step 6: Verify the data in YourDB_PITR, then extract or swap with production
USE [YourDB_PITR];
SELECT COUNT(*) FROM dbo.Orders WHERE order_date >= '2024-01-15';
```

---

## Scenario 3: Cross-Server Restore (Clone Database to Another Server)

Use when: cloning production to test/dev, migrating database to a new server.

```sql
-- On source server: take a copy-only backup (does not affect differential chain)
BACKUP DATABASE [ProductionDB]
TO DISK = N'\\FileShare\Backups\ProductionDB_Clone.bak'
WITH COPY_ONLY, COMPRESSION, CHECKSUM, STATS = 10;

-- On target server: get file names from backup
RESTORE FILELISTONLY FROM DISK = N'\\FileShare\Backups\ProductionDB_Clone.bak';
-- LogicalName         PhysicalName              Type
-- ProductionDB        C:\Data\ProductionDB.mdf  D
-- ProductionDB_log    C:\Log\ProductionDB_log.ldf L

-- Restore to target server with new paths
RESTORE DATABASE [DevDB]
FROM DISK = N'\\FileShare\Backups\ProductionDB_Clone.bak'
WITH
    MOVE N'ProductionDB'     TO N'D:\Data\DevDB.mdf',
    MOVE N'ProductionDB_log' TO N'D:\Log\DevDB_log.ldf',
    RECOVERY, REPLACE, STATS = 10;

-- Post-restore: fix orphaned users (logins may not exist on target)
USE [DevDB];
SELECT dp.name AS user_name
FROM sys.database_principals dp
LEFT JOIN sys.server_principals sp ON dp.sid = sp.sid
WHERE dp.type IN ('S', 'U')
  AND dp.name NOT IN ('dbo', 'guest', 'INFORMATION_SCHEMA', 'sys')
  AND sp.name IS NULL;

-- Fix orphaned users
ALTER USER [AppUser] WITH LOGIN = [AppUser];

-- Or drop and recreate users if logins differ on target
DROP USER [AppUser];
CREATE USER [AppUser] FOR LOGIN [DevAppLogin];
```

---

## Scenario 4: Fixing a Broken Log Chain

Use when: log backups have a gap (a full or copy-only backup was taken without your knowledge, breaking the chain).

**Symptom:** `RESTORE LOG` fails with "The log in this backup set begins at LSN X which is too recent to apply to the database."

```sql
-- Step 1: Identify the break — check LSN chain
SELECT
    CASE type WHEN 'D' THEN 'Full' WHEN 'I' THEN 'Diff' WHEN 'L' THEN 'Log' END AS type,
    backup_start_date,
    first_lsn,
    last_lsn,
    database_backup_lsn,
    physical_device_name
FROM msdb.dbo.backupset bs
JOIN msdb.dbo.backupmediafamily bmf ON bs.media_set_id = bmf.media_set_id
WHERE database_name = 'YourDB'
ORDER BY backup_start_date;
-- Look for a gap: a log backup whose first_lsn does not follow the previous last_lsn

-- Step 2: Identify the last valid full backup that starts a complete chain
-- After the break, find the most recent full backup whose last_lsn > the break point

-- Step 3: Re-start the restore chain from that full backup
RESTORE DATABASE [YourDB]
FROM DISK = N'D:\Backups\YourDB_Full_AfterBreak.bak'
WITH NORECOVERY, REPLACE, STATS = 10;

-- Step 4: Apply all log backups in order from that point
RESTORE LOG [YourDB] FROM DISK = N'D:\Backups\YourDB_Log_After1.trn' WITH NORECOVERY;
-- Continue until most recent log
RESTORE DATABASE [YourDB] WITH RECOVERY;

-- Prevention: use COPY_ONLY for ad hoc backups
-- BACKUP DATABASE [YourDB] TO DISK = '...' WITH COPY_ONLY;
```

---

## Scenario 5: Backup Verification and Integrity Check

Use when: validating backup files before a maintenance window, confirming offsite backup health.

```sql
-- Verify backup file integrity without performing a full restore
RESTORE VERIFYONLY
FROM DISK = N'D:\Backups\YourDB_Full_20240115.bak'
WITH CHECKSUM;
-- Success: "The backup set on file 1 is valid."
-- Failure: detailed error about corruption location

-- Check contents of the backup
RESTORE HEADERONLY FROM DISK = N'D:\Backups\YourDB_Full_20240115.bak';
-- Confirms: DatabaseName, BackupType, BackupStartDate, DatabaseVersion, Collation

-- Restore to a test location and run DBCC CHECKDB (most thorough verification)
RESTORE DATABASE [YourDB_VerifyTest]
FROM DISK = N'D:\Backups\YourDB_Full_20240115.bak'
WITH RECOVERY, REPLACE,
     MOVE N'YourDB'     TO N'D:\Temp\VerifyTest.mdf',
     MOVE N'YourDB_log' TO N'D:\Temp\VerifyTest_log.ldf';

DBCC CHECKDB ([YourDB_VerifyTest]) WITH NO_INFOMSGS, ALL_ERRORMSGS;
-- No errors = backup is fully consistent and restorable

-- Clean up test database
DROP DATABASE [YourDB_VerifyTest];
```
