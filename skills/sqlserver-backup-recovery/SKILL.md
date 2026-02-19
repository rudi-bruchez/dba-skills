---
name: sqlserver-backup-recovery
description: Designs SQL Server backup strategies and guides restore operations including full, differential, and log backups, point-in-time recovery, and backup verification. Use when designing backup strategies, performing restores, troubleshooting backup failures, or verifying backup integrity.
metadata:
  author: dba-skills
  version: "1.0"
  platform: SQL Server 2016+
---

# SQL Server Backup and Recovery

## Recovery Model Selection

Choose the recovery model before designing any backup strategy. Changing the recovery model mid-stream breaks the log chain.

```sql
-- Check current recovery models
SELECT name, recovery_model_desc, log_reuse_wait_desc
FROM sys.databases
WHERE database_id > 4;

-- Set recovery model
ALTER DATABASE [YourDB] SET RECOVERY FULL;        -- Production databases with RPO requirements
ALTER DATABASE [YourDB] SET RECOVERY BULK_LOGGED; -- During ETL windows only
ALTER DATABASE [YourDB] SET RECOVERY SIMPLE;      -- Dev/QA, no point-in-time recovery needed
```

**Recovery model decision guide:**

| Requirement | Recovery Model | Reasoning |
|---|---|---|
| RPO in minutes or seconds | FULL | Enables log backups for point-in-time recovery |
| Log shipping or Always On AG | FULL | Required by these technologies |
| ETL with large bulk loads | BULK_LOGGED (temporary) | Reduces log growth during bulk ops; switch back to FULL after |
| Dev/QA, no RPO requirement | SIMPLE | Log auto-truncated; no log backup overhead |
| Read-only reporting database | SIMPLE | No transactions to protect |

---

## Backup Strategy Design

Match backup frequency to your RPO and RTO requirements.

| RPO / RTO Target | Strategy | Backup Schedule |
|---|---|---|
| RPO 0, RTO < 30 sec | Always On AG synchronous | AG handles this; offload log backups to secondary |
| RPO 15 min, RTO 1 hr | Full + Differential + Log | Full weekly, Diff daily, Log every 15 min |
| RPO 1 hr, RTO 4 hr | Full + Log | Full weekly, Log every hour |
| RPO 24 hr, RTO 8 hr | Full only (SIMPLE) | Full nightly |
| RPO 1 week, RTO 24 hr | Full weekly (SIMPLE) | Full weekly, offsite copy |

**Retention recommendation:**
- Full backups: 4 weeks
- Differential backups: 2 weeks
- Log backups: 1 week (or longer based on compliance)

---

## Key Backup Commands

### Full backup
```sql
BACKUP DATABASE [YourDB]
TO DISK = N'D:\Backups\YourDB_Full_20240115.bak'
WITH
    COMPRESSION,
    CHECKSUM,
    STATS = 10,
    NAME = N'YourDB Full Backup';
```

### Differential backup
```sql
BACKUP DATABASE [YourDB]
TO DISK = N'D:\Backups\YourDB_Diff_20240115_1800.bak'
WITH DIFFERENTIAL, COMPRESSION, CHECKSUM, STATS = 10;
```

### Transaction log backup
```sql
BACKUP LOG [YourDB]
TO DISK = N'D:\Backups\YourDB_Log_20240115_1800.trn'
WITH COMPRESSION, CHECKSUM, STATS = 10;
```

### Copy-only backup (does not break differential chain)
```sql
BACKUP DATABASE [YourDB]
TO DISK = N'D:\Backups\YourDB_CopyOnly.bak'
WITH COPY_ONLY, COMPRESSION, CHECKSUM, STATS = 10;
```

### Striped backup (faster for large databases)
```sql
BACKUP DATABASE [YourDB]
TO DISK = N'D:\Backups\YourDB_Full_1.bak',
   DISK = N'E:\Backups\YourDB_Full_2.bak',
   DISK = N'F:\Backups\YourDB_Full_3.bak'
WITH COMPRESSION, CHECKSUM, STATS = 10;
```

**Always include COMPRESSION and CHECKSUM.** Enable compression as default at instance level:
```sql
EXEC sp_configure 'backup compression default', 1;
RECONFIGURE;
```

---

## Backup Verification

### Verify backup file integrity (no restore required)
```sql
RESTORE VERIFYONLY
FROM DISK = N'D:\Backups\YourDB_Full_20240115.bak'
WITH CHECKSUM;
```

### Inspect backup contents before restoring
```sql
-- List all backup sets in the file
RESTORE HEADERONLY FROM DISK = N'D:\Backups\YourDB_Full_20240115.bak';

-- List logical file names (needed for MOVE during restore)
RESTORE FILELISTONLY FROM DISK = N'D:\Backups\YourDB_Full_20240115.bak';

-- Show media label and checksum info
RESTORE LABELONLY FROM DISK = N'D:\Backups\YourDB_Full_20240115.bak';
```

---

## Point-in-Time Recovery Workflow

Use this workflow to restore a database to a specific point in time.

**Step 1 — Take a tail-log backup** (capture uncommitted transactions before restore)
```sql
BACKUP LOG [YourDB]
TO DISK = N'D:\Backups\YourDB_TailLog.trn'
WITH NO_TRUNCATE, NORECOVERY, CHECKSUM;
-- NO_TRUNCATE allows backup even if database is inaccessible
```

**Step 2 — Restore the most recent full backup** (leave in NORECOVERY state)
```sql
RESTORE DATABASE [YourDB]
FROM DISK = N'D:\Backups\YourDB_Full_20240115.bak'
WITH NORECOVERY, REPLACE, STATS = 10;
```

**Step 3 — Restore the most recent differential backup** (if available)
```sql
RESTORE DATABASE [YourDB]
FROM DISK = N'D:\Backups\YourDB_Diff_20240115_1800.bak'
WITH NORECOVERY, STATS = 10;
```

**Step 4 — Restore log backups in chronological order** until reaching the target time
```sql
RESTORE LOG [YourDB]
FROM DISK = N'D:\Backups\YourDB_Log_20240115_1900.trn'
WITH NORECOVERY, STATS = 10;

RESTORE LOG [YourDB]
FROM DISK = N'D:\Backups\YourDB_Log_20240115_2000.trn'
WITH NORECOVERY, STATS = 10;
```

**Step 5 — Apply the final log backup with STOPAT** (point-in-time)
```sql
RESTORE LOG [YourDB]
FROM DISK = N'D:\Backups\YourDB_Log_20240115_2100.trn'
WITH RECOVERY, STOPAT = N'2024-01-15 20:45:00';
-- RECOVERY brings the database online; no more logs can be applied after this
```

### Restore to a different database name or server
```sql
RESTORE DATABASE [YourDB_Test]
FROM DISK = N'D:\Backups\YourDB_Full_20240115.bak'
WITH
    MOVE N'YourDB_Data' TO N'D:\Data\YourDB_Test.mdf',
    MOVE N'YourDB_Log'  TO N'D:\Log\YourDB_Test_log.ldf',
    RECOVERY, REPLACE, STATS = 10;
-- Get logical file names from RESTORE FILELISTONLY first
```

---

## Backup History Monitoring Queries

### Recent backup history
```sql
SELECT TOP 50
    bs.database_name,
    CASE bs.type
        WHEN 'D' THEN 'Full'
        WHEN 'I' THEN 'Differential'
        WHEN 'L' THEN 'Log'
    END AS backup_type,
    bs.backup_start_date,
    bs.backup_finish_date,
    DATEDIFF(MINUTE, bs.backup_start_date, bs.backup_finish_date) AS duration_min,
    bs.backup_size / 1024.0 / 1024.0 / 1024.0            AS size_gb,
    bs.compressed_backup_size / 1024.0 / 1024.0 / 1024.0 AS compressed_gb,
    bmf.physical_device_name AS backup_file
FROM msdb.dbo.backupset bs
JOIN msdb.dbo.backupmediafamily bmf ON bs.media_set_id = bmf.media_set_id
ORDER BY bs.backup_start_date DESC;
```

### Databases without recent full backup (alert if > 25 hours)
```sql
SELECT
    d.name AS database_name,
    d.recovery_model_desc,
    MAX(bs.backup_finish_date)                                      AS last_full_backup,
    DATEDIFF(HOUR, MAX(bs.backup_finish_date), GETDATE())           AS hours_since_backup
FROM sys.databases d
LEFT JOIN msdb.dbo.backupset bs
    ON d.name = bs.database_name AND bs.type = 'D'
WHERE d.database_id > 4
GROUP BY d.name, d.recovery_model_desc
HAVING MAX(bs.backup_finish_date) IS NULL
    OR MAX(bs.backup_finish_date) < DATEADD(HOUR, -25, GETDATE())
ORDER BY hours_since_backup DESC;
```

### FULL recovery databases missing log backups (alert if > 60 minutes)
```sql
SELECT
    d.name AS database_name,
    MAX(bs.backup_finish_date)                                      AS last_log_backup,
    DATEDIFF(MINUTE, MAX(bs.backup_finish_date), GETDATE())         AS minutes_since_log_backup
FROM sys.databases d
LEFT JOIN msdb.dbo.backupset bs
    ON d.name = bs.database_name AND bs.type = 'L'
WHERE d.database_id > 4
  AND d.recovery_model_desc = 'FULL'
  AND d.state = 0
GROUP BY d.name
HAVING MAX(bs.backup_finish_date) IS NULL
    OR MAX(bs.backup_finish_date) < DATEADD(MINUTE, -60, GETDATE())
ORDER BY minutes_since_log_backup DESC;
```

---

## Common Backup/Restore Issues

| Error | Cause | Fix |
|---|---|---|
| Log growing uncontrollably | FULL recovery, no log backups | `SELECT log_reuse_wait_desc FROM sys.databases` — value 'LOG_BACKUP' means take a log backup immediately |
| "Media family error" | Backup file already exists with different media set | Use `WITH INIT` to overwrite, or delete old file |
| "Backup of a database other than the existing" | Restoring over wrong database | Add `WITH REPLACE` to the RESTORE command |
| LSN chain broken | Full backup taken out-of-band broke the chain | Identify the last valid full backup via HEADERONLY and restart chain from there |
| `log_reuse_wait_desc = 'REPLICATION'` | Transactional replication not cleaned up | Remove replication or run `sp_repldone` |

---

## References

- [Backup commands](references/backup-commands.md) — complete T-SQL backup and restore commands with all options
- [Restore scenarios](references/restore-scenarios.md) — step-by-step procedures for each restore scenario
- [Backup monitoring](references/backup-monitoring.md) — monitoring queries for backup health and history
- [Examples](examples/examples.md) — full restore, PITR, cross-server restore, broken log chain fix, backup verification
