# PostgreSQL Backup & Recovery Research

## Overview
PostgreSQL backup and recovery involve creating data dumps, physical cluster backups, and managing transaction logs (WAL) to ensure durability and allow point-in-time recovery.

## Backup Types
- **Logical Backup (pg_dump):** Dumps individual databases or schemas into SQL commands or custom archive formats. Flexible for version upgrades or single database restores.
- **Physical Backup (pg_basebackup):** Creates a byte-for-byte copy of the entire database cluster. Essential for setting up replicas and performing point-in-time recovery.
- **Continuous Archiving (WAL):** Regularly archiving Write-Ahead Log (WAL) segments to a separate location (e.g., S3 or a separate server).

## Point-in-Time Recovery (PITR)
- **Mechanism:** Replaying WAL segments over a physical base backup.
- **Benefits:** Restoring to a specific moment (e.g., just before a bad `DELETE`).
- **Required Settings:** `wal_level = replica` and `archive_mode = on`.
- **`recovery.conf` (pre-PG 12) / `postgresql.auto.conf` (PG 12+):** Used to specify the recovery target and location of WAL archives.

## Backup Verification & Integrity
- **Checksums:** Enable data checksums when initializing the cluster (`initdb -k`) to detect data corruption.
- **Restore Testing:** Regularly test the restoration process to ensure backups and WAL segments are valid and consistent.

## Disaster Recovery Strategies
- **RPO (Recovery Point Objective):** Aim for RPO near zero using streaming replication or synchronous replication.
- **RTO (Recovery Time Objective):** Minimize using hot standbys and automated failover tools.
- **Offsite Storage:** Store backups and WAL segments in geographically separate locations (e.g., S3, Google Cloud Storage).

## Best Practices (2024-2025)
- **Enterprise-Grade Tools:**
    - **pgBackRest:** Highly optimized physical backup and recovery with compression, encryption, and parallel processing.
    - **Barman (Backup and Recovery Manager):** Comprehensive tool for centralized backup management.
- **Incremental Backups:** Use pgBackRest or other tools for faster backups of large databases.
- **Managed Backup:** Utilize RDS or Cloud SQL for managed backup/recovery if applicable.
- **Retention Policies:** Define how long to keep base backups and WAL archives to balance storage and recovery needs.

## Key Shell Commands
- `pg_dump -U [user] -d [db_name] -F c -f db_backup.dump`
- `pg_restore -U [user] -d [db_name] db_backup.dump`
- `pg_basebackup -h [host] -D [backup_dir] -U [user] -P --wal-method=stream`

## References
- [PostgreSQL Documentation: Backup and Restore](https://www.postgresql.org/docs/current/backup.html)
- [pgBackRest: Documentation](https://pgbackrest.org/user-guide.html)
- [Barman: Documentation](https://www.pgbarman.org/documentation/)
