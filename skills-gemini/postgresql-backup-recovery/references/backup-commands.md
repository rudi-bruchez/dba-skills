# Backup Commands Guide

A comprehensive list of common PostgreSQL backup commands and their options.

## 1. Logical Backup (pg_dump)
Dumps a single database into a SQL script or a custom archive format.
```bash
# Custom format (flexible for pg_restore)
pg_dump -U [user] -d [db_name] -F c -f db_backup.dump

# SQL format (human-readable)
pg_dump -U [user] -d [db_name] -F p -f db_backup.sql

# Backup a specific schema
pg_dump -U [user] -d [db_name] -n [schema_name] -F c -f schema_backup.dump
```

## 2. Cluster-Wide Logical Backup (pg_dumpall)
Dumps the entire database cluster (all databases and roles).
```bash
pg_dumpall -U [user] -F p -f cluster_backup.sql
```

## 3. Physical Backup (pg_basebackup)
Creates a byte-for-byte copy of the entire database cluster.
```bash
pg_basebackup -h [host] -D [backup_dir] -U [user] -P --wal-method=stream --format=p --gzip
```
- **-h [host]:** The primary server's hostname or IP address.
- **-D [backup_dir]:** The target directory for the backup files.
- **-U [user]:** The replication user to perform the backup.
- **-P:** Shows progress during the backup.
- **--wal-method=stream:** Streams Write-Ahead Log (WAL) segments to the backup directory.
- **--format=p:** Use the plain (directory) format.
- **--gzip:** Compresses the backup files.

## 4. Continuous Archiving (WAL)
To enable WAL archiving, configure `postgresql.conf`:
```conf
wal_level = replica
archive_mode = on
archive_command = 'cp %p /path/to/archive/%f'
```
- **%p:** The path to the WAL file to archive.
- **%f:** The name of the WAL file.

## References
- [PostgreSQL Documentation: pg_dump](https://www.postgresql.org/docs/current/app-pgdump.html)
- [PostgreSQL Documentation: pg_basebackup](https://www.postgresql.org/docs/current/app-pgbasebackup.html)
- [PostgreSQL Documentation: Continuous Archiving](https://www.postgresql.org/docs/current/continuous-archiving.html)
