# SQL Server Backup Monitoring Queries

---

## Recent Backup History

```sql
-- All backup activity in the last 7 days
SELECT TOP 100
    bs.database_name,
    CASE bs.type
        WHEN 'D' THEN 'Full'
        WHEN 'I' THEN 'Differential'
        WHEN 'L' THEN 'Log'
        WHEN 'F' THEN 'File'
        WHEN 'G' THEN 'Filegroup'
        WHEN 'P' THEN 'Partial'
    END AS backup_type,
    bs.backup_start_date,
    bs.backup_finish_date,
    DATEDIFF(MINUTE, bs.backup_start_date, bs.backup_finish_date) AS duration_min,
    bs.backup_size / 1024.0 / 1024.0 / 1024.0                    AS size_gb,
    bs.compressed_backup_size / 1024.0 / 1024.0 / 1024.0         AS compressed_gb,
    CAST(100.0 * bs.compressed_backup_size / NULLIF(bs.backup_size,0) AS DECIMAL(5,2))
        AS compression_pct,
    bs.is_copy_only,
    bs.has_backup_checksums,
    bmf.physical_device_name                                       AS backup_file
FROM msdb.dbo.backupset bs
JOIN msdb.dbo.backupmediafamily bmf ON bs.media_set_id = bmf.media_set_id
WHERE bs.backup_start_date >= DATEADD(DAY, -7, GETDATE())
ORDER BY bs.backup_start_date DESC;
```

---

## Backup Coverage Checks (Alerting Queries)

### Databases without a full backup in the last 25 hours
```sql
SELECT
    d.name                                                            AS database_name,
    d.recovery_model_desc,
    MAX(bs.backup_finish_date)                                        AS last_full_backup,
    DATEDIFF(HOUR, MAX(bs.backup_finish_date), GETDATE())             AS hours_since_backup,
    CASE WHEN MAX(bs.backup_finish_date) IS NULL THEN 'NEVER BACKED UP'
         ELSE 'OVERDUE' END                                           AS alert_reason
FROM sys.databases d
LEFT JOIN msdb.dbo.backupset bs
    ON d.name = bs.database_name AND bs.type = 'D'
WHERE d.database_id > 4       -- Exclude system databases
  AND d.state = 0             -- Online only
  AND d.source_database_id IS NULL  -- Exclude snapshots
GROUP BY d.name, d.recovery_model_desc
HAVING MAX(bs.backup_finish_date) IS NULL
    OR MAX(bs.backup_finish_date) < DATEADD(HOUR, -25, GETDATE())
ORDER BY hours_since_backup DESC;
```

### FULL recovery databases without a log backup in the last 60 minutes
```sql
SELECT
    d.name                                                            AS database_name,
    MAX(bs.backup_finish_date)                                        AS last_log_backup,
    DATEDIFF(MINUTE, MAX(bs.backup_finish_date), GETDATE())           AS minutes_since_log_backup
FROM sys.databases d
LEFT JOIN msdb.dbo.backupset bs
    ON d.name = bs.database_name AND bs.type = 'L'
WHERE d.database_id > 4
  AND d.recovery_model_desc = 'FULL'
  AND d.state = 0
  AND d.source_database_id IS NULL
GROUP BY d.name
HAVING MAX(bs.backup_finish_date) IS NULL
    OR MAX(bs.backup_finish_date) < DATEADD(MINUTE, -60, GETDATE())
ORDER BY minutes_since_log_backup DESC;
```

### Most recent backup for each database (dashboard view)
```sql
SELECT
    d.name                                                            AS database_name,
    d.recovery_model_desc,
    MAX(CASE WHEN bs.type = 'D' THEN bs.backup_finish_date END)       AS last_full,
    MAX(CASE WHEN bs.type = 'I' THEN bs.backup_finish_date END)       AS last_diff,
    MAX(CASE WHEN bs.type = 'L' THEN bs.backup_finish_date END)       AS last_log,
    DATEDIFF(HOUR, MAX(CASE WHEN bs.type = 'D' THEN bs.backup_finish_date END), GETDATE())
        AS hours_since_full
FROM sys.databases d
LEFT JOIN msdb.dbo.backupset bs ON d.name = bs.database_name
WHERE d.database_id > 4 AND d.state = 0 AND d.source_database_id IS NULL
GROUP BY d.name, d.recovery_model_desc
ORDER BY d.name;
```

---

## Backup Job History

```sql
-- SQL Server Agent backup job history (last 7 days)
SELECT
    j.name     AS job_name,
    jh.step_id,
    jh.step_name,
    CASE jh.run_status
        WHEN 0 THEN 'Failed'
        WHEN 1 THEN 'Succeeded'
        WHEN 2 THEN 'Retry'
        WHEN 3 THEN 'Cancelled'
    END AS run_status,
    msdb.dbo.agent_datetime(jh.run_date, jh.run_time) AS run_datetime,
    jh.run_duration,
    LEFT(jh.message, 500) AS message
FROM msdb.dbo.sysjobs j
JOIN msdb.dbo.sysjobhistory jh ON j.job_id = jh.job_id
WHERE j.name LIKE '%backup%' OR j.name LIKE '%Backup%'
  AND msdb.dbo.agent_datetime(jh.run_date, jh.run_time) >= DATEADD(DAY, -7, GETDATE())
ORDER BY run_datetime DESC;
```

---

## Log Space and Growth Monitoring

```sql
-- Log space usage in current database
SELECT
    DB_NAME()                   AS database_name,
    total_log_size_mb,
    used_log_space_mb,
    used_log_space_percent,
    log_backup_time             AS last_log_backup_time
FROM sys.dm_db_log_space_usage;
-- Alert: used_log_space_percent > 80%

-- Log reuse wait (why is the log not being truncated?)
SELECT name, log_reuse_wait_desc
FROM sys.databases
WHERE log_reuse_wait_desc <> 'NOTHING'
  AND database_id > 4;
-- LOG_BACKUP = log backup needed immediately
-- ACTIVE_TRANSACTION = long-running open transaction blocking truncation
-- REPLICATION = replication not cleaned up

-- VLF count (high VLF count = log file fragmentation)
SELECT DB_NAME(database_id) AS database_name, COUNT(*) AS vlf_count
FROM sys.dm_db_log_info(NULL)
GROUP BY database_id
ORDER BY vlf_count DESC;
-- Alert: vlf_count > 1000
```

---

## Backup Size and Duration Trends

```sql
-- Weekly backup duration trend (detect gradual slowdowns)
SELECT
    database_name,
    DATEPART(YEAR, backup_start_date)  AS backup_year,
    DATEPART(WEEK, backup_start_date)  AS backup_week,
    COUNT(*)                           AS backup_count,
    AVG(DATEDIFF(MINUTE, backup_start_date, backup_finish_date)) AS avg_duration_min,
    AVG(backup_size / 1024.0 / 1024 / 1024)  AS avg_size_gb,
    AVG(compressed_backup_size / 1024.0 / 1024 / 1024) AS avg_compressed_gb
FROM msdb.dbo.backupset
WHERE type = 'D'
  AND backup_start_date >= DATEADD(WEEK, -12, GETDATE())
GROUP BY database_name, DATEPART(YEAR, backup_start_date), DATEPART(WEEK, backup_start_date)
ORDER BY database_name, backup_year, backup_week;
```

---

## Restore Chain Validation

```sql
-- Validate LSN chain for a specific database (last 7 days)
SELECT
    database_name,
    CASE type WHEN 'D' THEN 'Full' WHEN 'I' THEN 'Diff' WHEN 'L' THEN 'Log' END AS backup_type,
    backup_start_date,
    backup_finish_date,
    first_lsn,
    last_lsn,
    checkpoint_lsn,
    database_backup_lsn,
    physical_device_name
FROM msdb.dbo.backupset bs
JOIN msdb.dbo.backupmediafamily bmf ON bs.media_set_id = bmf.media_set_id
WHERE bs.database_name = 'YourDB'
  AND bs.backup_start_date >= DATEADD(DAY, -7, GETDATE())
ORDER BY bs.backup_start_date;
-- For a valid log chain: each log backup's first_lsn should be <= previous backup's last_lsn
```

---

## Backup File Locations

```sql
-- All backup file paths (verify network paths are accessible)
SELECT DISTINCT
    bs.database_name,
    bmf.physical_device_name,
    bs.backup_start_date
FROM msdb.dbo.backupset bs
JOIN msdb.dbo.backupmediafamily bmf ON bs.media_set_id = bmf.media_set_id
WHERE bs.backup_start_date >= DATEADD(DAY, -1, GETDATE())
ORDER BY bs.backup_start_date DESC;
```
