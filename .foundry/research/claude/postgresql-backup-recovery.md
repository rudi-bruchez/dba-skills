# PostgreSQL Backup & Recovery

## Overview
PostgreSQL offers several backup strategies: logical backups (pg_dump), physical/file-system backups (pg_basebackup), and continuous archiving with WAL (Point-in-Time Recovery, PITR). A comprehensive strategy typically combines all three. Recovery Time Objective (RTO) and Recovery Point Objective (RPO) determine which methods to use.

---

## Backup Strategy Overview

| Method | Type | RPO | RTO | Consistency | Size |
|--------|------|-----|-----|-------------|------|
| `pg_dump` | Logical | Last backup | Minutes-hours | Per-dump snapshot | Compressed, smaller |
| `pg_dumpall` | Logical | Last backup | Minutes-hours | Cluster-wide snapshot | Larger |
| `pg_basebackup` | Physical | Last backup | Fast | Full cluster | Full data directory |
| PITR (basebackup + WAL) | Physical | Near-zero | Minutes | Exact point in time | Full + WAL files |
| Continuous WAL archiving | WAL | Seconds-minutes | Depends | Used with basebackup | WAL segments only |

---

## pg_dump — Logical Backups

### Basic Usage
```bash
# Dump a single database (custom format — recommended)
pg_dump -Fc -d mydb -f mydb_backup.dump

# Dump with verbose output
pg_dump -Fc -v -d mydb -f mydb_backup.dump

# Plain SQL format
pg_dump -Fp -d mydb -f mydb_backup.sql

# Directory format (parallel-friendly)
pg_dump -Fd -j 4 -d mydb -f mydb_backup_dir/

# Dump specific schema
pg_dump -Fc -n public -d mydb -f public_backup.dump

# Dump specific table(s)
pg_dump -Fc -t orders -t order_items -d mydb -f orders_backup.dump

# Exclude tables matching a pattern
pg_dump -Fc -T 'log_*' -d mydb -f mydb_no_logs.dump

# Dump schema only (no data)
pg_dump -Fc -s -d mydb -f mydb_schema.dump

# Dump data only
pg_dump -Fc -a -d mydb -f mydb_data.dump
```

### pg_dump Connection Options
```bash
pg_dump -h hostname -p 5432 -U username -d dbname -f backup.dump
# Use .pgpass or PGPASSWORD env var for password
export PGPASSWORD='mypassword'
# Or use a connection string
pg_dump "postgresql://user:pass@host:5432/dbname" -Fc -f backup.dump
```

### Restore with pg_restore
```bash
# Restore to existing (empty) database
pg_restore -d mydb -Fc backup.dump

# Restore with verbose + parallel jobs
pg_restore -d mydb -j 4 -v backup.dump

# Create database and restore
pg_restore -C -d postgres backup.dump   # -C creates the database

# Restore specific schema only
pg_restore -d mydb -n myschema backup.dump

# Restore specific table
pg_restore -d mydb -t orders backup.dump

# List contents of a dump file
pg_restore -l backup.dump

# Restore with selective objects using list file
pg_restore -l backup.dump > restore_list.txt
# Edit restore_list.txt, then:
pg_restore -d mydb -L restore_list.txt backup.dump

# Restore schema only
pg_restore -d mydb -s backup.dump

# Restore data only
pg_restore -d mydb -a backup.dump
```

### pg_dumpall — Full Cluster Backup
```bash
# Dump entire cluster (all databases + globals)
pg_dumpall -f cluster_backup.sql

# Dump globals only (roles, tablespaces)
pg_dumpall --globals-only -f globals.sql

# Restore cluster backup
psql -f cluster_backup.sql postgres

# Restore globals only
psql -f globals.sql postgres
```

---

## pg_basebackup — Physical Backup

### Basic Usage
```bash
# Basic basebackup (tar format, gzipped)
pg_basebackup -h localhost -U replicator -D /backup/basebackup \
  -Ft -z -P -Xs

# Parameters explained:
# -D  target directory
# -Ft tar format (Fp = plain files)
# -z  gzip compression
# -P  progress reporting
# -Xs WAL included via streaming (recommended)
# -Xf WAL via fetch (alternative)

# Basebackup with WAL streaming (recommended)
pg_basebackup -h localhost -U replicator \
  -D /backup/$(date +%Y%m%d_%H%M%S) \
  -Fp -Xs -P -R

# -R writes recovery.conf / postgresql.auto.conf for standby
# -Fp plain files (easier to inspect)

# Compressed tar with checkpoint
pg_basebackup -h localhost -U replicator \
  -D /backup/base.tar.gz \
  -Ft -z --checkpoint=fast -P -Xs

# With tablespace mapping
pg_basebackup -h localhost -U replicator \
  -D /backup/basebackup \
  -Fp -Xs -P \
  --tablespace-map=/original/tblspc=/backup/tblspc

# Label the backup
pg_basebackup -h localhost -U replicator \
  -D /backup/basebackup \
  -Ft -Xs -P \
  --label="nightly_$(date +%Y%m%d)"
```

### Required Permissions for pg_basebackup
```sql
-- Create a dedicated replication role
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'strong_password';

-- pg_hba.conf must allow replication connections:
-- host  replication  replicator  10.0.0.0/8  scram-sha-256
```

### Verifying a Basebackup
```bash
# List backup contents
ls -la /backup/basebackup/

# For tar format, test integrity
gzip -t /backup/basebackup/base.tar.gz && echo "OK"

# Verify backup label
cat /backup/basebackup/backup_label   # (for plain format)
tar -xOf /backup/basebackup/base.tar.gz backup_label  # (for tar format)
```

---

## WAL Archiving & PITR

### Configuring WAL Archiving
In `postgresql.conf`:
```ini
wal_level = replica               # or logical (required for archiving)
archive_mode = on                 # Enable archiving
archive_command = 'cp %p /wal_archive/%f'  # Copy to local directory
# Or use rsync:
archive_command = 'rsync -a %p backup@server:/wal_archive/%f'
# Or use pgBackRest/WAL-G for production

archive_timeout = 300             # Force WAL segment switch every 5 min (limits RPO)

# Verify archive command succeeded (exit code 0 = success)
# %p = path to WAL file, %f = filename only
```

### Using WAL-G (Production-grade WAL Archiving)
```bash
# Archive command using WAL-G with S3
archive_command = 'wal-g wal-push %p'

# Restore command
restore_command = 'wal-g wal-fetch %f %p'

# Create a basebackup with WAL-G
wal-g backup-push /var/lib/postgresql/data

# List backups
wal-g backup-list

# Restore specific backup
wal-g backup-fetch /var/lib/postgresql/data LATEST
```

### Using pgBackRest (Enterprise-grade)
```bash
# Initialize stanza (repository configuration)
pgbackrest --stanza=main stanza-create

# Full backup
pgbackrest --stanza=main --type=full backup

# Differential backup
pgbackrest --stanza=main --type=diff backup

# Incremental backup
pgbackrest --stanza=main --type=incr backup

# List backups
pgbackrest --stanza=main info

# Restore latest backup
pgbackrest --stanza=main restore

# PITR restore to specific time
pgbackrest --stanza=main --type=time \
  --target="2025-06-15 14:30:00" restore
```

---

## Point-in-Time Recovery (PITR)

### Concepts
- **WAL LSN** (Log Sequence Number): unique identifier for each WAL record
- **Recovery target**: can be a time, XID, LSN, or named restore point
- **Timeline**: branch identifier; each recovery creates a new timeline
- **pg_wal_replay_pause()** / **pg_wal_replay_resume()**: pause/resume standby replay

### Creating a Named Restore Point
```sql
-- Create a named restore point (superuser required)
SELECT pg_create_restore_point('before_migration_20250615');
```

### PITR Recovery Configuration

**For PostgreSQL 12+**, create `postgresql.conf` additions or `postgresql.auto.conf`:
```ini
restore_command = 'cp /wal_archive/%f %p'
# or for WAL-G:
restore_command = 'wal-g wal-fetch %f %p'

# Recovery target options (choose one):
recovery_target_time = '2025-06-15 14:30:00 UTC'
recovery_target_xid = '12345678'
recovery_target_lsn = '0/15D5A50'
recovery_target_name = 'before_migration_20250615'
recovery_target = 'immediate'   # Recover to a consistent state ASAP

# What to do after reaching target
recovery_target_action = 'promote'    # Promote to primary (default)
recovery_target_action = 'pause'      # Pause for inspection
recovery_target_action = 'shutdown'   # Shutdown after recovery

# Whether to include the target transaction
recovery_target_inclusive = true      # Include the target point (default)
```

**Create the signal file** (PostgreSQL 12+):
```bash
touch /var/lib/postgresql/data/recovery.signal
```

**Pre-PostgreSQL 12**, use `recovery.conf` file in data directory (no signal file needed).

### PITR Recovery Procedure
```bash
# 1. Stop PostgreSQL
pg_ctl stop -D /var/lib/postgresql/data

# 2. Move or backup the corrupted data directory
mv /var/lib/postgresql/data /var/lib/postgresql/data.corrupted

# 3. Restore the base backup
pg_basebackup -D /var/lib/postgresql/data ...
# or restore from pg_basebackup tar:
mkdir /var/lib/postgresql/data
tar -xzf base.tar.gz -C /var/lib/postgresql/data

# 4. Configure recovery (postgresql.conf or postgresql.auto.conf)
cat >> /var/lib/postgresql/data/postgresql.auto.conf << EOF
restore_command = 'cp /wal_archive/%f %p'
recovery_target_time = '2025-06-15 14:30:00 UTC'
recovery_target_action = 'promote'
EOF

# 5. Create recovery signal
touch /var/lib/postgresql/data/recovery.signal

# 6. Fix ownership
chown -R postgres:postgres /var/lib/postgresql/data

# 7. Start PostgreSQL — it will enter recovery mode
pg_ctl start -D /var/lib/postgresql/data

# 8. Monitor recovery progress
tail -f /var/log/postgresql/postgresql.log
# Look for: "recovery stopping before commit..."
# And: "database system is ready to accept connections"
```

---

## Monitoring Backup Operations

### Check WAL Archiving Status
```sql
-- Current archiving status
SELECT archived_count,
       last_archived_wal,
       last_archived_time,
       failed_count,
       last_failed_wal,
       last_failed_time,
       stats_reset
FROM pg_stat_archiver;
```
**Alert:** `failed_count > 0` or `last_failed_time` is recent.

### Check WAL Files in pg_wal
```bash
# Count WAL files (should be relatively stable)
ls -1 /var/lib/postgresql/data/pg_wal/ | wc -l

# Size of pg_wal directory
du -sh /var/lib/postgresql/data/pg_wal/
```

### Check Backup Status from SQL
```sql
-- Current WAL LSN
SELECT pg_current_wal_lsn();

-- WAL segment file name
SELECT pg_walfile_name(pg_current_wal_lsn());

-- Backup state (if backup is in progress)
SELECT * FROM pg_stat_activity WHERE backend_type = 'walsender';

-- Age of the oldest backup file needed
SELECT pg_walfile_name(pg_current_wal_lsn()),
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), '0/0'::pg_lsn)) AS total_wal;
```

---

## Backup Validation & Testing

### Test Logical Backup Integrity
```bash
# Verify dump is readable
pg_restore -l mydb_backup.dump > /dev/null && echo "Dump is valid"

# Restore to a test instance
pg_restore -h testhost -d test_mydb mydb_backup.dump

# Compare row counts
psql -d mydb -c "SELECT count(*) FROM orders;"
psql -d test_mydb -c "SELECT count(*) FROM orders;"
```

### Test Physical Backup Recovery
```bash
# Restore to a different host/port to verify
pg_basebackup -D /tmp/test_recovery ...
# Start PostgreSQL on a different port
pg_ctl start -D /tmp/test_recovery -o "-p 5433"
# Verify data
psql -p 5433 -d mydb -c "SELECT count(*) FROM orders;"
```

---

## Key Configuration Parameters

| Parameter | Purpose | Recommendation |
|-----------|---------|----------------|
| `wal_level` | WAL content level | `replica` minimum; `logical` for logical replication |
| `archive_mode` | Enable WAL archiving | `on` for PITR capability |
| `archive_command` | Command to archive WAL | Use WAL-G or pgBackRest in production |
| `archive_timeout` | Max seconds before forced WAL switch | 60-300s (limits RPO) |
| `max_wal_senders` | Max WAL streaming connections | >= 3 (standbys + backup) |
| `wal_keep_size` | Keep this much WAL for standbys | 1024 MB+ depending on lag |
| `checkpoint_completion_target` | Spread checkpoint I/O | 0.9 |
| `max_wal_size` | Trigger checkpoint at this WAL size | 1-4GB |

---

## Best Practices

1. **3-2-1 Rule**: 3 copies of data, 2 different media, 1 offsite.
2. **Test restores regularly** — at least monthly, preferably weekly for critical databases.
3. **Use `pg_dump -Fc`** (custom format) for single-database backups — supports parallel restore and selective object restore.
4. **Use pgBackRest or WAL-G** for production — they handle compression, encryption, S3/GCS storage, and verify backups.
5. **Monitor `pg_stat_archiver`** — `failed_count > 0` means backups are silently failing.
6. **Set `archive_timeout`** to bound your RPO — without it, a quiet database might not archive WAL for hours.
7. **Document recovery procedures** and practice them — a backup you can't restore is useless.
8. **Encrypt backups** — use `pg_dump | gpg` or pgBackRest's built-in encryption.
9. **Version-match tools** — use the same major version of `pg_dump`/`pg_restore` as the server.
10. **Store backup metadata** (start/end LSN, timestamp, database version) alongside each backup.

---

## Common Issues & Solutions

| Issue | Cause | Solution |
|-------|-------|---------|
| pg_dump hangs | Lock contention on tables | Use `--lock-wait-timeout` option |
| Archive command fails silently | Disk full, permission error | Monitor `pg_stat_archiver.failed_count` |
| WAL accumulates in pg_wal | Archive falling behind | Investigate archive_command, add storage |
| PITR stops before target time | Insufficient WAL | Ensure all WAL segments are in archive |
| pg_basebackup slow | Large database, slow network | Use `-Ft -z`, run during low-traffic periods |
| Recovery timeline confusion | Multiple recovery attempts | Each recovery creates new timeline; check `pg_control` |
| pg_restore fails with FK errors | Dependencies between tables | Use `--disable-triggers` or restore to empty schema |
