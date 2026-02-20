# Backup Monitoring

Essential queries to monitor backup history, health, and configuration.

## 1. View Last Backup Dates for All Databases
Identify databases that haven't been backed up recently.
```sql
SELECT
    d.name AS DatabaseName,
    MAX(b.backup_finish_date) AS LastBackupDate,
    DATEDIFF(day, MAX(b.backup_finish_date), GETDATE()) AS DaysSinceLastBackup,
    b.type AS BackupType -- D=Full, I=Differential, L=Log
FROM sys.databases AS d
LEFT JOIN msdb.dbo.backupset AS b ON d.name = b.database_name
GROUP BY d.name, b.type
ORDER BY DaysSinceLastBackup DESC;
```

## 2. Check Database Recovery Models
Ensure production databases are using the correct recovery model.
```sql
SELECT name, recovery_model_desc
FROM sys.databases
WHERE name NOT IN ('master', 'model', 'msdb', 'tempdb');
```

## 3. Monitor Active Backup/Restore Tasks
Check the progress and estimated completion time for active operations.
```sql
SELECT
    session_id,
    command,
    percent_complete,
    start_time,
    DATEADD(ms, estimated_completion_time, GETDATE()) AS EstimatedCompletionTime,
    (estimated_completion_time / 1000 / 60) AS MinutesToFinish
FROM sys.dm_exec_requests
WHERE command IN ('BACKUP DATABASE', 'RESTORE DATABASE', 'BACKUP LOG', 'RESTORE LOG');
```

## 4. Identify Suspect Pages
Detect database pages with corruption that may require recovery.
```sql
SELECT database_id, file_id, page_id, event_type, last_update_date
FROM msdb.dbo.suspect_pages;
```

## 5. View Backup History and Sizes
```sql
SELECT
    database_name,
    backup_start_date,
    backup_finish_date,
    DATEDIFF(minute, backup_start_date, backup_finish_date) AS BackupDuration_min,
    backup_size / 1024 / 1024 AS BackupSize_MB,
    compressed_backup_size / 1024 / 1024 AS CompressedSize_MB
FROM msdb.dbo.backupset
ORDER BY backup_finish_date DESC;
```

## Best Practices
- **Automated Alerts:** Set up SQL Agent alerts for failed backups and long-running backup tasks.
- **Reporting:** Regularly report on backup status and RPO compliance to stakeholders.
- **Integrity Checks:** Regularly run `DBCC CHECKDB` to ensure the source data is healthy before taking backups.
