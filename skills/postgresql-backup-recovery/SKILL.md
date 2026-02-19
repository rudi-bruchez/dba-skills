---
name: postgresql-backup-recovery
description: Manages PostgreSQL backup and recovery using pg_dump, pg_basebackup, WAL archiving, and point-in-time recovery (PITR). Use when designing backup strategies, performing logical or physical backups, recovering from data loss, or setting up continuous WAL archiving for PITR capability.
metadata:
  author: dba-skills
  version: "1.0"
  platform: PostgreSQL 13+
---

# PostgreSQL Backup and Recovery

## Backup Strategy Selection

Choose the backup method based on Recovery Point Objective (RPO) and Recovery Time Objective (RTO):

| Method | RPO | RTO | Use Case |
|--------|-----|-----|---------|
| `pg_dump` (logical) | Last backup | Minutes–hours | Single database, schema changes, cross-version migration |
| `pg_dumpall` (logical) | Last backup | Minutes–hours | Full cluster including roles and tablespaces |
| `pg_basebackup` (physical) | Last backup | Fast (minutes) | Base for standby; faster restore than logical |
| `pg_basebackup + WAL archiving` | Seconds–minutes | Minutes | PITR capability; production databases |
| WAL-G / pgBackRest | Near-zero | Minutes | Production; S3/cloud storage; incremental |

**Rule of thumb:**
- Development / small databases: `pg_dump -Fc` nightly
- Production databases requiring PITR: `pg_basebackup + WAL archiving`
- Enterprise production: pgBackRest or WAL-G with cloud storage

---

## Logical Backups: pg_dump

Logical backups capture database contents as SQL or binary. They are database-version-independent for restore.

### Essential pg_dump Commands

```bash
# Custom format (recommended): supports parallel restore and selective object restore
pg_dump -Fc -d mydb -f mydb_$(date +%Y%m%d_%H%M%S).dump

# With verbose output to monitor progress
pg_dump -Fc -v -d mydb -f mydb_backup.dump

# Directory format for parallel backup (4 jobs)
pg_dump -Fd -j 4 -d mydb -f mydb_backup_dir/

# Backup specific tables only
pg_dump -Fc -t orders -t order_items -d mydb -f orders_backup.dump

# Exclude tables matching a pattern
pg_dump -Fc -T 'log_*' -T 'audit_*' -d mydb -f mydb_no_logs.dump

# Schema only (no data)
pg_dump -Fc -s -d mydb -f mydb_schema.dump

# Dump all databases including roles and tablespaces
pg_dumpall -f cluster_backup.sql
pg_dumpall --globals-only -f globals_roles.sql
```

### Restore Commands

```bash
# Restore to existing empty database
pg_restore -d mydb -Fc mydb_backup.dump

# Restore with parallel jobs (use with directory format or custom format)
pg_restore -d mydb -j 4 -v mydb_backup.dump

# Create database then restore in one step
pg_restore -C -d postgres mydb_backup.dump

# Restore only specific tables
pg_restore -d mydb -t orders mydb_backup.dump

# List dump contents before restoring
pg_restore -l mydb_backup.dump

# Restore cluster backup
psql -f cluster_backup.sql postgres
```

### pg_dump Options Reference

| Option | Effect |
|--------|--------|
| `-Fc` | Custom binary format (recommended) |
| `-Fd` | Directory format (enables parallel with `-j`) |
| `-Fp` | Plain SQL text |
| `-j N` | Parallel jobs (directory format only for dump; any format for restore) |
| `-s` | Schema only (no data) |
| `-a` | Data only (no schema) |
| `-t table` | Dump specific table (repeatable) |
| `-T table` | Exclude table (repeatable, supports wildcards) |
| `-n schema` | Dump specific schema |
| `-N schema` | Exclude schema |
| `--lock-wait-timeout=N` | Fail if can't acquire lock within N ms |
| `--exclude-table-data=t` | Include table in schema but not its data |

---

## Physical Backups: pg_basebackup

Physical backups copy the entire data directory. They are faster to restore and are the foundation for standby servers and PITR.

### Setup: Required Role and pg_hba.conf

```sql
-- Create a dedicated role for backups
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'strong_backup_password';
```

```
# pg_hba.conf: allow replication connections
host  replication  replicator  10.0.0.0/8  scram-sha-256
host  replication  replicator  127.0.0.1/32  scram-sha-256
```

### Essential pg_basebackup Commands

```bash
# Tar format, gzipped, with progress, streaming WAL (recommended)
pg_basebackup \
    -h localhost -U replicator \
    -D /backup/$(date +%Y%m%d_%H%M%S) \
    -Ft -z -P -Xs

# Plain files format for easier inspection and restore
pg_basebackup \
    -h localhost -U replicator \
    -D /backup/$(date +%Y%m%d_%H%M%S) \
    -Fp -Xs -P

# With -R flag: automatically writes standby.signal and postgresql.auto.conf
# for use as a streaming replication standby
pg_basebackup \
    -h primary.example.com -U replicator \
    -D /var/lib/postgresql/data \
    -Fp -Xs -P -R

# Verify the backup label
cat /backup/20250619_020000/backup_label
# or for tar format:
tar -xOf /backup/base.tar.gz backup_label
```

| pg_basebackup option | Effect |
|---------------------|--------|
| `-D dir` | Target directory |
| `-Ft` | Tar format |
| `-Fp` | Plain files format |
| `-z` | Gzip compression (with `-Ft`) |
| `-P` | Progress reporting |
| `-Xs` | Include WAL via streaming (recommended) |
| `-Xf` | Include WAL via fetch (simpler, no extra slot) |
| `-R` | Write `standby.signal` + `primary_conninfo` for standby use |
| `--checkpoint=fast` | Request immediate checkpoint instead of waiting |

---

## WAL Archiving Configuration

WAL archiving enables PITR by continuously copying WAL segments to an archive location.

### postgresql.conf Settings

```ini
wal_level = replica            # Minimum: replica (or logical for logical replication)
archive_mode = on              # Enable WAL archiving (requires restart)
archive_command = 'cp %p /wal_archive/%f'  # Simple local copy
# %p = full path to WAL file, %f = filename only

# For remote archive via rsync:
# archive_command = 'rsync -a %p backup@archive-server:/wal_archive/%f'

# Force WAL segment switch every 5 minutes (limits RPO for quiet databases)
archive_timeout = 300

# Required WAL settings for standbys and PITR:
max_wal_senders = 5            # Connections for WAL streaming (standbys + backup)
wal_keep_size = 1024           # MB of WAL to retain in pg_wal for lagging standbys
```

**For production environments, use WAL-G or pgBackRest instead of `cp`:**

```ini
# WAL-G with S3:
archive_command = 'wal-g wal-push %p'

# pgBackRest:
archive_command = 'pgbackrest --stanza=main archive-push %p'
```

### Monitor Archive Status

```sql
SELECT archived_count,
       last_archived_wal,
       last_archived_time,
       failed_count,
       last_failed_wal,
       last_failed_time,
       now() - last_archived_time AS time_since_last_archive
FROM pg_stat_archiver;
```

**Alert on:** `failed_count > 0` or `time_since_last_archive > 10 minutes` on an active database.

---

## Point-in-Time Recovery (PITR) Procedure

PITR restores a base backup and replays WAL to reach a specific point in time.

### Pre-Recovery: Create a Named Restore Point (Optional)

```sql
-- Run this before a risky operation (migration, bulk update)
SELECT pg_create_restore_point('before_migration_20250619');
```

### PITR Recovery Steps (PostgreSQL 12+)

```bash
# Step 1: Stop PostgreSQL
pg_ctl stop -D /var/lib/postgresql/data
# or: systemctl stop postgresql

# Step 2: Move the damaged data directory to a safe location
mv /var/lib/postgresql/data /var/lib/postgresql/data.damaged

# Step 3: Restore the base backup
mkdir /var/lib/postgresql/data
tar -xzf /backup/base.tar.gz -C /var/lib/postgresql/data
# or if using plain format: cp -a /backup/20250619_020000/. /var/lib/postgresql/data/

# Step 4: Configure recovery in postgresql.auto.conf
cat >> /var/lib/postgresql/data/postgresql.auto.conf << 'EOF'
restore_command = 'cp /wal_archive/%f %p'
recovery_target_time = '2025-06-19 14:30:00 UTC'
recovery_target_action = 'promote'
recovery_target_inclusive = true
EOF

# Step 5: Create the recovery signal file (PostgreSQL 12+)
touch /var/lib/postgresql/data/recovery.signal

# Step 6: Fix ownership
chown -R postgres:postgres /var/lib/postgresql/data
chmod 700 /var/lib/postgresql/data

# Step 7: Start PostgreSQL — it enters recovery mode automatically
pg_ctl start -D /var/lib/postgresql/data
# or: systemctl start postgresql

# Step 8: Monitor recovery progress
tail -f /var/log/postgresql/postgresql.log
# Look for:
#   LOG: starting point-in-time recovery
#   LOG: restored log file "000000010000000000000001" from archive
#   LOG: recovery stopping before commit...
#   LOG: database system is ready to accept read only connections
#   LOG: database system is ready to accept connections  (after promote)
```

### Recovery Target Options

```ini
# Recover to a specific timestamp
recovery_target_time = '2025-06-19 14:30:00 UTC'

# Recover to just before a specific transaction ID
recovery_target_xid = '12345678'

# Recover to a specific LSN
recovery_target_lsn = '0/15D5A50'

# Recover to a named restore point
recovery_target_name = 'before_migration_20250619'

# Recover to a consistent state as quickly as possible (for hardware failures)
recovery_target = 'immediate'

# What to do after reaching target:
recovery_target_action = 'promote'   # Open for writes (default)
recovery_target_action = 'pause'     # Pause for inspection before promoting
recovery_target_action = 'shutdown'  # Shutdown to prevent unintended promotion
```

---

## Backup Verification

**A backup that has never been tested is not a backup.**

### Verify Logical Backup Integrity

```bash
# Check that the dump file is readable (does not fully restore)
pg_restore -l mydb_backup.dump > /dev/null && echo "Dump is valid"

# Full test restore to a separate instance
createdb test_restore
pg_restore -d test_restore -j 4 mydb_backup.dump

# Compare row counts
psql -d mydb -c "SELECT count(*) FROM orders;"
psql -d test_restore -c "SELECT count(*) FROM orders;"

# Clean up test database
dropdb test_restore
```

### Verify Physical Backup

```bash
# Test by starting the restored backup on a different port
pg_ctl start -D /backup/test_recovery -o "-p 5433"
psql -p 5433 -d mydb -c "SELECT count(*) FROM orders;"
pg_ctl stop -D /backup/test_recovery
```

---

## Key Configuration Parameters Summary

| Parameter | Purpose | Recommended Value |
|-----------|---------|-----------------|
| `wal_level` | WAL content level | `replica` (minimum for PITR) |
| `archive_mode` | Enable WAL archiving | `on` for production |
| `archive_command` | Archive WAL segments | `cp` for simple; WAL-G/pgBackRest for production |
| `archive_timeout` | Force WAL segment switch | 300 seconds (5 minutes) |
| `max_wal_senders` | Streaming connections | 5 or more |
| `wal_keep_size` | Retain WAL for standbys | 1024 MB minimum |
| `checkpoint_completion_target` | Spread checkpoint I/O | 0.9 |

---

## Reference Files

- [Backup commands reference](references/backup-commands.md) — complete pg_dump, pg_restore, pg_basebackup options
- [PITR setup guide](references/pitr-setup.md) — WAL archiving configuration and PITR walkthrough
- [Recovery scenarios](references/recovery-scenarios.md) — step-by-step recovery procedures for common failure scenarios

## Examples

- [Backup and recovery scenarios](examples/examples.md) — full backup/restore workflow, physical backup setup, WAL archiving, PITR, cross-version migration
