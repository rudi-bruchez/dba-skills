# SQL Server Backup and Recovery — Examples

---

## Scenario 1: Full Database Restore After Hardware Failure

**Context:** The data drive on `SQL-PROD-01` failed at 3:42 AM. The database `OrdersDB` is unavailable. The last successful backup history in `msdb` was replicated to `SQL-PROD-02` (monitoring server).

**Available backups:**
- Full: Sunday 11 PM → `\\BackupShare\OrdersDB_Full_20240114_2300.bak`
- Differential: Monday 11 PM → `\\BackupShare\OrdersDB_Diff_20240115_2300.bak`
- Log backups: every 15 minutes, last one at 3:30 AM → `\\BackupShare\OrdersDB_Log_20240116_0330.trn`

**Step 1 — Verify backup files are accessible**
```sql
RESTORE VERIFYONLY FROM DISK = N'\\BackupShare\OrdersDB_Full_20240114_2300.bak' WITH CHECKSUM;
RESTORE VERIFYONLY FROM DISK = N'\\BackupShare\OrdersDB_Diff_20240115_2300.bak' WITH CHECKSUM;
-- Both return: "The backup set on file 1 is valid."
```

**Step 2 — Get logical file names**
```sql
RESTORE FILELISTONLY FROM DISK = N'\\BackupShare\OrdersDB_Full_20240114_2300.bak';
-- Returns: OrdersDB (data), OrdersDB_log (log)
```

**Step 3 — Restore sequence on replacement server**
```sql
-- Full backup
RESTORE DATABASE [OrdersDB]
FROM DISK = N'\\BackupShare\OrdersDB_Full_20240114_2300.bak'
WITH NORECOVERY, REPLACE,
     MOVE N'OrdersDB'     TO N'E:\Data\OrdersDB.mdf',
     MOVE N'OrdersDB_log' TO N'F:\Log\OrdersDB_log.ldf',
     STATS = 10;

-- Differential
RESTORE DATABASE [OrdersDB]
FROM DISK = N'\\BackupShare\OrdersDB_Diff_20240115_2300.bak'
WITH NORECOVERY, STATS = 10;

-- Log backups (apply in order — 23:15, 23:30, ... through 03:30)
RESTORE LOG [OrdersDB] FROM DISK = N'\\BackupShare\OrdersDB_Log_20240115_2315.trn' WITH NORECOVERY;
-- ... (apply each 15-minute log) ...
RESTORE LOG [OrdersDB] FROM DISK = N'\\BackupShare\OrdersDB_Log_20240116_0330.trn' WITH NORECOVERY;

-- Bring online
RESTORE DATABASE [OrdersDB] WITH RECOVERY;
```

**Result:** Database restored with 12 minutes of data loss (3:30 AM last log backup; failure at 3:42 AM). RPO achieved: 15 minutes.

---

## Scenario 2: Point-in-Time Recovery — Accidental DELETE

**Context:** A developer ran `DELETE FROM dbo.OrderItems WHERE order_id = 5000` in production instead of the test database. The mistake was noticed 22 minutes later. The database is in FULL recovery model with 15-minute log backups.

**Step 1 — Identify the exact time of the mistake**
```sql
-- Check the SQL Server error log for connection activity
EXEC sys.xp_readerrorlog 0, 1, N'Login', NULL, NULL, NULL, N'desc';
-- Also check extended events if LongRunningQueries session is active
```

The developer confirms they ran the statement at approximately 14:23.

**Step 2 — Identify backup chain**
```sql
SELECT CASE type WHEN 'D' THEN 'Full' WHEN 'I' THEN 'Diff' WHEN 'L' THEN 'Log' END AS type,
       backup_start_date, backup_finish_date,
       physical_device_name
FROM msdb.dbo.backupset bs
JOIN msdb.dbo.backupmediafamily bmf ON bs.media_set_id = bmf.media_set_id
WHERE database_name = 'OrdersDB'
  AND backup_start_date >= DATEADD(HOUR, -12, GETDATE())
ORDER BY backup_start_date;
-- Full: 23:00 last night
-- Log: 14:00, 14:15, 14:30 (the 14:30 backup contains the DELETE)
```

**Step 3 — Restore to a parallel database (do not overwrite production)**
```sql
-- Restore full backup
RESTORE DATABASE [OrdersDB_PITR]
FROM DISK = N'D:\Backups\OrdersDB_Full_20240115_2300.bak'
WITH NORECOVERY, REPLACE,
     MOVE N'OrdersDB'     TO N'D:\Temp\OrdersDB_PITR.mdf',
     MOVE N'OrdersDB_log' TO N'D:\Temp\OrdersDB_PITR_log.ldf';

-- Apply all log backups through 14:15 (last one before the mistake)
RESTORE LOG [OrdersDB_PITR] FROM DISK = N'D:\Backups\OrdersDB_Log_20240116_1400.trn' WITH NORECOVERY;

-- Apply the 14:30 log backup with STOPAT one second before the mistake
RESTORE LOG [OrdersDB_PITR]
FROM DISK = N'D:\Backups\OrdersDB_Log_20240116_1415.trn'
WITH RECOVERY, STOPAT = N'2024-01-16 14:22:59';
```

**Step 4 — Extract and re-insert the deleted data**
```sql
-- On OrdersDB_PITR: export the missing rows
SELECT * FROM OrdersDB_PITR.dbo.OrderItems WHERE order_id = 5000;
-- 142 rows returned

-- Re-insert into production
INSERT INTO OrdersDB.dbo.OrderItems
SELECT * FROM OrdersDB_PITR.dbo.OrderItems WHERE order_id = 5000;
```

**Step 5 — Clean up**
```sql
DROP DATABASE [OrdersDB_PITR];
```

**Result:** 142 rows recovered. Total data loss: zero. Time to recovery: 35 minutes.

---

## Scenario 3: Cross-Server Restore — Refreshing Test Environment

**Context:** The QA team needs an up-to-date copy of `ProductionCRM` restored to the test server `SQL-TEST-01`. This is done monthly.

```sql
-- On production: take a copy-only backup (does not affect differential chain)
BACKUP DATABASE [ProductionCRM]
TO DISK = N'\\SharedBackup\ProductionCRM_Clone_20240115.bak'
WITH COPY_ONLY, COMPRESSION, CHECKSUM, STATS = 10;

-- On SQL-TEST-01: get logical file names
RESTORE FILELISTONLY FROM DISK = N'\\SharedBackup\ProductionCRM_Clone_20240115.bak';

-- Restore as TestCRM (different name to coexist with other test databases)
RESTORE DATABASE [TestCRM]
FROM DISK = N'\\SharedBackup\ProductionCRM_Clone_20240115.bak'
WITH
    MOVE N'ProductionCRM'     TO N'D:\TestData\TestCRM.mdf',
    MOVE N'ProductionCRM_log' TO N'D:\TestLog\TestCRM_log.ldf',
    RECOVERY, REPLACE, STATS = 10;

-- Post-restore cleanup: obfuscate sensitive data in test environment
USE [TestCRM];
-- Mask customer email addresses
UPDATE dbo.Customers SET email = 'test_' + CAST(customer_id AS VARCHAR) + '@test.internal';
-- Reset all passwords to a known test value
UPDATE dbo.Users SET password_hash = HASHBYTES('SHA2_256', 'TestPassword123!');

-- Fix orphaned users from production logins that do not exist on test
SELECT dp.name AS orphaned_user
FROM sys.database_principals dp
LEFT JOIN sys.server_principals sp ON dp.sid = sp.sid
WHERE dp.type IN ('S', 'U')
  AND dp.name NOT IN ('dbo', 'guest', 'INFORMATION_SCHEMA', 'sys')
  AND sp.name IS NULL;
-- Fix each: ALTER USER [username] WITH LOGIN = [test_login];
```

---

## Scenario 4: Fixing a Broken Log Chain

**Context:** A junior DBA ran a full backup of `FinanceDB` manually (without `COPY_ONLY`) at 2 PM during a maintenance window. Now the 3 PM log backup restore fails with "The log in this backup set begins at LSN X, which is too recent to apply to the database."

**Diagnosis:**
```sql
-- Check LSN chain
SELECT CASE type WHEN 'D' THEN 'Full' WHEN 'I' THEN 'Diff' WHEN 'L' THEN 'Log' END AS type,
       backup_start_date, first_lsn, last_lsn, database_backup_lsn
FROM msdb.dbo.backupset
WHERE database_name = 'FinanceDB'
  AND backup_start_date >= DATEADD(HOUR, -6, GETDATE())
ORDER BY backup_start_date;
-- 11:00 PM: Full (last_lsn = 1000)
-- 2:00 PM:  Full (manual, non-COPY_ONLY — resets differential base, last_lsn = 5000)
-- 2:15 PM:  Log  (first_lsn = 5001 — belongs to the 2 PM full's chain)
-- 3:00 PM:  Log  (first_lsn = 5501)
-- The 11 PM restore chain cannot accept logs starting at 5001
```

**Resolution:**
The only valid restore path now uses the 2 PM full backup as the starting point.

```sql
-- Take a tail-log backup of current state (if database is accessible)
BACKUP LOG [FinanceDB]
TO DISK = N'D:\Backups\FinanceDB_TailLog_Emergency.trn'
WITH NO_TRUNCATE, NORECOVERY, CHECKSUM;

-- Restore from the 2 PM full backup
RESTORE DATABASE [FinanceDB]
FROM DISK = N'D:\Backups\FinanceDB_Full_20240116_1400.bak'
WITH NORECOVERY, REPLACE, STATS = 10;

-- Apply all logs from 2:15 PM onward
RESTORE LOG [FinanceDB] FROM DISK = N'D:\Backups\FinanceDB_Log_20240116_1415.trn' WITH NORECOVERY;
RESTORE LOG [FinanceDB] FROM DISK = N'D:\Backups\FinanceDB_Log_20240116_1430.trn' WITH NORECOVERY;
-- ... (apply remaining logs) ...
RESTORE LOG [FinanceDB] FROM DISK = N'D:\Backups\FinanceDB_TailLog_Emergency.trn' WITH NORECOVERY;
RESTORE DATABASE [FinanceDB] WITH RECOVERY;
```

**Prevention:** Always use `WITH COPY_ONLY` for ad hoc backups.
```sql
-- Safe ad hoc backup that does not affect the chain
BACKUP DATABASE [FinanceDB] TO DISK = N'...' WITH COPY_ONLY, COMPRESSION, CHECKSUM;
```

---

## Scenario 5: Backup Verification Before Decommissioning

**Context:** Before decommissioning a storage array, verify that all database backups stored on it are valid and restorable.

```sql
-- Step 1: List all backup files on the storage array
SELECT DISTINCT bmf.physical_device_name, bs.database_name, bs.backup_start_date
FROM msdb.dbo.backupmediafamily bmf
JOIN msdb.dbo.backupset bs ON bmf.media_set_id = bs.media_set_id
WHERE bmf.physical_device_name LIKE 'D:\OldArray\%'  -- Filter by array path
ORDER BY bs.database_name, bs.backup_start_date DESC;

-- Step 2: Verify each backup file
RESTORE VERIFYONLY FROM DISK = N'D:\OldArray\OrdersDB_Full_20240115.bak' WITH CHECKSUM;
RESTORE VERIFYONLY FROM DISK = N'D:\OldArray\FinanceDB_Full_20240115.bak' WITH CHECKSUM;
-- Repeat for each backup file

-- Step 3: For critical databases, do a test restore (most reliable verification)
RESTORE DATABASE [OrdersDB_VerifyOnly]
FROM DISK = N'D:\OldArray\OrdersDB_Full_20240115.bak'
WITH RECOVERY, REPLACE,
     MOVE N'OrdersDB'     TO N'C:\Temp\VerifyTest.mdf',
     MOVE N'OrdersDB_log' TO N'C:\Temp\VerifyTest_log.ldf';

-- Run database consistency check
DBCC CHECKDB ([OrdersDB_VerifyOnly]) WITH NO_INFOMSGS, ALL_ERRORMSGS;
-- "DBCC results for 'OrdersDB_VerifyOnly'. ... CHECKDB found 0 allocation errors and 0 consistency errors"

-- Clean up after verification
DROP DATABASE [OrdersDB_VerifyOnly];

-- Step 4: Copy verified backups to new storage before decommissioning
-- robocopy D:\OldArray \\NewStorage\Backups /COPYALL /MIR
```
