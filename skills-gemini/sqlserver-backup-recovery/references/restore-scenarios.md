# Restore Scenarios

Common SQL Server restore scenarios and their recovery procedures.

## 1. Full Database Restore
Restore a database from its full backup.
```sql
RESTORE DATABASE [MyDatabase]
FROM DISK = 'C:\Backups\MyDatabase_Full.bak'
WITH RECOVERY, REPLACE, STATS = 10;
```
- **RECOVERY:** Makes the database available for use after the restore.
- **REPLACE:** Overwrites the existing database if it exists.

## 2. Restore with NORECOVERY (Point-in-Time Restore)
Restore the full backup first, keeping it in a "Restoring" state to allow for subsequent differential or log restores.
```sql
-- 1. Restore Full Backup (NORECOVERY)
RESTORE DATABASE [MyDatabase]
FROM DISK = 'C:\Backups\MyDatabase_Full.bak'
WITH NORECOVERY, REPLACE;

-- 2. Restore Differential Backup (NORECOVERY)
RESTORE DATABASE [MyDatabase]
FROM DISK = 'C:\Backups\MyDatabase_Diff.bak'
WITH NORECOVERY;

-- 3. Restore Transaction Log(s) (RECOVERY)
RESTORE LOG [MyDatabase]
FROM DISK = 'C:\Backups\MyDatabase_Log.trn'
WITH RECOVERY;
```

## 3. Restore to a Point-in-Time
Use the `STOPAT` clause to recover to a specific moment.
```sql
-- 1. Restore Full Backup (NORECOVERY)
RESTORE DATABASE [MyDatabase]
FROM DISK = 'C:\Backups\MyDatabase_Full.bak'
WITH NORECOVERY, REPLACE;

-- 2. Restore Transaction Log(s) (RECOVERY, STOPAT)
RESTORE LOG [MyDatabase]
FROM DISK = 'C:\Backups\MyDatabase_Log.trn'
WITH STOPAT = '2023-10-27 14:30:00', RECOVERY;
```

## 4. Restore with Page-Level Recovery
Recover a specific data page from a backup file (Enterprise Edition only).
```sql
-- 1. Identify the damaged page
SELECT database_id, file_id, page_id, error_type, last_update_date
FROM msdb.dbo.suspect_pages;

-- 2. Restore the specific page
RESTORE DATABASE [MyDatabase]
PAGE = '1:1234' -- file:page
FROM DISK = 'C:\Backups\MyDatabase_Full.bak'
WITH NORECOVERY;

-- 3. Restore subsequent transaction log backups
RESTORE LOG [MyDatabase]
FROM DISK = 'C:\Backups\MyDatabase_Log.trn'
WITH RECOVERY;
```

## 5. Verify Backup Integrity
```sql
RESTORE VERIFYONLY
FROM DISK = 'C:\Backups\MyDatabase_Full.bak';
```
- **VERIFYONLY:** Checks the backup file's headers and readability without actually restoring the data.

## References
- [Microsoft Documentation: RESTORE (Transact-SQL)](https://learn.microsoft.com/en-us/sql/t-sql/statements/restore-statements-transact-sql)
- [Microsoft Documentation: Point-in-Time Restore](https://learn.microsoft.com/en-us/sql/relational-databases/backup-restore/restore-a-sql-server-database-to-a-point-in-time-full-recovery-model)
