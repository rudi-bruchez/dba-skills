---
name: sqlserver-backup-recovery
description: Expert skill for managing SQL Server backup strategies, performing restores, and ensuring data durability and recovery.
version: 1.0.0
tags:
  - sqlserver
  - dba
  - backup
  - recovery
  - disaster-recovery
---

# SQL Server Backup & Recovery

This skill provides expert guidance for designing, implementing, and managing SQL Server backup and recovery strategies to protect against data loss and minimize downtime.

## Core Capabilities

- **Backup Strategy Design:** Choosing between Simple, Full, and Bulk-Logged recovery models.
- **Backup Operations:** Performing Full, Differential, and Transaction Log backups.
- **Recovery Procedures:** Restoring databases to a point-in-time or a specific backup.
- **Integrity & Verification:** Ensuring backups are valid and restorable.
- **Disaster Recovery (DR):** Implementing strategies for offsite storage and business continuity.

## Workflow: Backup Strategy

1.  **Define RPO & RTO:** Determine the maximum tolerable data loss (RPO) and downtime (RTO).
2.  **Select Recovery Model:**
    - **Full:** For point-in-time recovery and maximum durability.
    - **Simple:** For test/dev environments or read-only databases where log backups aren't needed.
3.  **Schedule Backups:**
    - Weekly or Daily Full backups.
    - Daily or Hourly Differential backups.
    - Frequent (e.g., 15-minute) Transaction Log backups (if using Full recovery).
4.  **Enable Verification:** Always use `WITH CHECKSUM` and perform regular `RESTORE VERIFYONLY` tests.
5.  **Offsite Storage:** Store backups in a separate location (e.g., Azure Blob Storage, S3, or a secondary data center).

## Essential Commands

### 1. Perform a Full Backup
```sql
BACKUP DATABASE [MyDatabase]
TO DISK = 'C:\Backups\MyDatabase_Full.bak'
WITH CHECKSUM, COMPRESSION, STATS = 10;
```

### 2. Perform a Transaction Log Backup
```sql
BACKUP LOG [MyDatabase]
TO DISK = 'C:\Backups\MyDatabase_Log.trn'
WITH CHECKSUM, COMPRESSION, STATS = 10;
```

### 3. Point-in-Time Restore (PITR)
Restore a database to a specific moment in time.
```sql
-- 1. Restore the Full Backup
RESTORE DATABASE [MyDatabase]
FROM DISK = 'C:\Backups\MyDatabase_Full.bak'
WITH NORECOVERY, REPLACE;

-- 2. Restore the Transaction Log(s) up to the target time
RESTORE LOG [MyDatabase]
FROM DISK = 'C:\Backups\MyDatabase_Log.trn'
WITH STOPAT = '2023-10-27 14:30:00', RECOVERY;
```

### 4. Verify a Backup
```sql
RESTORE VERIFYONLY
FROM DISK = 'C:\Backups\MyDatabase_Full.bak';
```

## Best Practices (2024-2025)

- **Backup to URL:** Direct backup to cloud storage (Azure Blob Storage) using a SAS token.
- **Immutable Storage:** Store backups in immutable storage to protect against ransomware.
- **Ola Hallengren's Solution:** Use the community standard maintenance scripts for automated backups.
- **SQL Server 2025:** Leverage Zstandard (ZSTD) backup compression for better performance.
- **Regular Restore Testing:** Automate the process of restoring backups to a non-production environment regularly.

## References

- [Backup Commands Guide](./references/backup-commands.md)
- [Restore Scenarios](./references/restore-scenarios.md)
- [Backup Monitoring](./references/backup-monitoring.md)
---
