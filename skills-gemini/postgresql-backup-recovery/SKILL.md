---
name: postgresql-backup-recovery
description: Expert skill for managing PostgreSQL backup strategies, performing restores, and ensuring data durability and point-in-time recovery (PITR).
version: 1.0.0
tags:
  - postgresql
  - dba
  - backup
  - recovery
  - disaster-recovery
---

# PostgreSQL Backup & Recovery

This skill provides expert guidance for designing, implementing, and managing PostgreSQL backup and recovery strategies to protect against data loss and minimize downtime.

## Core Capabilities

- **Backup Type Selection:** Choosing between logical (pg_dump) and physical (pg_basebackup) backups.
- **Continuous Archiving:** Managing Write-Ahead Logs (WAL) for point-in-time recovery.
- **Restore Operations:** Performing single database restores and full cluster recoveries.
- **Integrity & Verification:** Ensuring backups are valid and restorable.
- **Disaster Recovery (DR):** Implementing strategies for offsite storage and business continuity.

## Workflow: Backup Strategy

1.  **Define RPO & RTO:** Determine the maximum tolerable data loss (RPO) and downtime (RTO).
2.  **Select Backup Method:**
    - **Logical (pg_dump):** For small databases, individual schemas, or version upgrades.
    - **Physical (pg_basebackup):** For full cluster backups, replication setup, and PITR.
3.  **Enable WAL Archiving:** Set `wal_level = replica` and `archive_mode = on` for point-in-time recovery.
4.  **Schedule Backups:**
    - Weekly or Daily Physical backups (`pg_basebackup`).
    - Frequent (e.g., 5-minute) WAL archiving.
5.  **Enable Checksums:** Initialize the cluster with `initdb -k` to detect data corruption.
6.  **Offsite Storage:** Store backups and WAL archives in a separate location (e.g., S3, Google Cloud Storage).

## Essential Commands

### 1. Logical Backup (pg_dump)
```bash
pg_dump -U [user] -d [db_name] -F c -f db_backup.dump
```
- **-F c:** Use the custom archive format (most flexible).

### 2. Physical Backup (pg_basebackup)
```bash
pg_basebackup -h [host] -D [backup_dir] -U [user] -P --wal-method=stream
```
- **--wal-method=stream:** Streams WAL segments during the backup.

### 3. Logical Restore (pg_restore)
```bash
pg_restore -U [user] -d [db_name] db_backup.dump
```

### 4. Point-in-Time Recovery (PITR)
Requires base backup and WAL archives.
```bash
# 1. Stop the PostgreSQL server
# 2. Extract the base backup into the data directory
# 3. Create a recovery.signal file (PG 12+)
touch /var/lib/postgresql/data/recovery.signal
# 4. Configure restore_command and recovery_target_time in postgresql.conf or postgresql.auto.conf
# 5. Start the PostgreSQL server
```

## Best Practices (2024-2025)

- **Use pgBackRest:** Highly optimized physical backup and recovery with compression, encryption, and parallel processing.
- **Continuous Archiving:** Always enable WAL archiving as a second line of defense for DR and PITR.
- **Verify Backups Regularly:** Perform "restore tests" to ensure backups and WAL segments are valid.
- **Immutable Storage:** Store backups in immutable storage to protect against ransomware.
- **Managed Backup:** Utilize RDS or Cloud SQL for managed backup/recovery if applicable.

## References

- [Backup Commands Guide](./references/backup-commands.md)
- [PITR Setup Guide](./references/pitr-setup.md)
- [Recovery Scenarios](./references/recovery-scenarios.md)
