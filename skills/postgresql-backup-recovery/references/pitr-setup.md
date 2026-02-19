# PostgreSQL PITR Setup Guide

## How PITR Works

1. A **base backup** captures the entire data directory at a point in time.
2. **WAL segments** are continuously archived as the database runs.
3. During recovery, PostgreSQL restores the base backup, then replays WAL segments up to the recovery target.
4. Each recovery creates a new **timeline** to prevent replaying WAL from the wrong branch.

## WAL Archiving Configuration

### Step 1 — postgresql.conf Settings

```ini
# Minimum WAL content for archiving and replication
wal_level = replica           # 'logical' if logical replication is also needed

# Enable WAL archiving (requires server restart)
archive_mode = on

# Command run for each WAL segment; must return exit code 0 on success
# %p = absolute path to WAL file
# %f = filename only
archive_command = 'cp %p /wal_archive/%f'

# Force WAL segment to close and archive every N seconds
# Limits RPO for quiet databases that don't fill WAL segments quickly
archive_timeout = 300         # 5 minutes

# Ensure enough WAL senders for standbys + backup operations
max_wal_senders = 5

# Keep WAL in pg_wal in case standby falls behind
wal_keep_size = 1024          # 1 GB
```

After editing `postgresql.conf`:
```bash
# archive_mode requires restart
systemctl restart postgresql
# or
pg_ctl restart -D /var/lib/postgresql/data
```

### Step 2 — Create Archive Directory

```bash
mkdir -p /wal_archive
chown postgres:postgres /wal_archive
chmod 700 /wal_archive
```

### Step 3 — Test Archiving

```sql
-- Force a WAL switch to test the archive_command
SELECT pg_switch_wal();

-- Check that archiving succeeded
SELECT archived_count, last_archived_wal, last_archived_time,
       failed_count, last_failed_wal, last_failed_time
FROM pg_stat_archiver;
```

Verify the archive file was created:
```bash
ls -la /wal_archive/
```

### Production Archive Commands

For production, use WAL-G or pgBackRest instead of `cp`:

```ini
# WAL-G with S3
archive_command = 'wal-g wal-push %p'

# pgBackRest
archive_command = 'pgbackrest --stanza=main archive-push %p'

# rsync to remote server (simple but no compression/encryption)
archive_command = 'rsync -a %p backup-server:/wal_archive/%f'
```

---

## Taking a Base Backup for PITR

```bash
# Take a base backup (include WAL via streaming)
pg_basebackup \
    -h localhost \
    -U replicator \
    -D /backup/base_$(date +%Y%m%d_%H%M%S) \
    -Ft -z \          # tar + gzip
    -Xs \             # WAL via streaming
    -P \              # progress
    --checkpoint=fast \
    --label="pitr_base_$(date +%Y%m%d)"

# Verify backup label is present
tar -xOf /backup/base_20250619_020000/base.tar.gz backup_label
```

---

## Performing PITR Recovery (PostgreSQL 12+)

### Full Procedure

```bash
# ============================================================
# Step 1: Stop the database server
# ============================================================
systemctl stop postgresql
# or
pg_ctl stop -D /var/lib/postgresql/data -m fast

# ============================================================
# Step 2: Preserve the damaged data directory
# ============================================================
mv /var/lib/postgresql/data /var/lib/postgresql/data.$(date +%Y%m%d_%H%M%S)
# Do NOT delete it yet — keep as reference and for potential rollback

# ============================================================
# Step 3: Restore the most recent base backup before the target time
# ============================================================
mkdir -p /var/lib/postgresql/data
chmod 700 /var/lib/postgresql/data
chown postgres:postgres /var/lib/postgresql/data

# If tar format:
tar -xzf /backup/base_20250619_020000/base.tar.gz -C /var/lib/postgresql/data

# If tablespace tar files exist, restore those too:
# tar -xzf /backup/base_20250619_020000/pg_tblspc_*.tar.gz -C /tablespace/path/

# ============================================================
# Step 4: Configure recovery target
# ============================================================
# Append to postgresql.auto.conf (or add to postgresql.conf)
cat >> /var/lib/postgresql/data/postgresql.auto.conf << 'EOF'
restore_command = 'cp /wal_archive/%f %p'
recovery_target_time = '2025-06-19 14:30:00 UTC'
recovery_target_action = 'promote'
recovery_target_inclusive = true
EOF

# ============================================================
# Step 5: Create recovery signal file (PostgreSQL 12+)
# ============================================================
touch /var/lib/postgresql/data/recovery.signal
chown postgres:postgres /var/lib/postgresql/data/recovery.signal

# ============================================================
# Step 6: Fix ownership and permissions
# ============================================================
chown -R postgres:postgres /var/lib/postgresql/data
chmod 700 /var/lib/postgresql/data

# ============================================================
# Step 7: Start PostgreSQL — recovery begins automatically
# ============================================================
systemctl start postgresql
# or
pg_ctl start -D /var/lib/postgresql/data

# ============================================================
# Step 8: Monitor recovery progress
# ============================================================
tail -f /var/log/postgresql/postgresql-$(date +%Y-%m-%d).log
```

**Expected log messages during recovery:**
```
LOG:  starting point-in-time recovery to 2025-06-19 14:30:00+00
LOG:  restored log file "000000010000000000000001" from archive
LOG:  redo starts at 0/1000028
...
LOG:  recovery stopping before commit of transaction 12345678, time 2025-06-19 14:30:05+00
LOG:  pausing at the end of recovery
LOG:  database system is ready to accept read only connections
# (after promote action:)
LOG:  database system is ready to accept connections
```

### Recovery After Promote

```sql
-- Verify the database is writable (recovery signal file is removed automatically)
SELECT pg_is_in_recovery();   -- Should return false after promotion

-- Check recovery timeline
SELECT timeline_id FROM pg_control_checkpoint();

-- Verify data integrity
SELECT count(*) FROM critical_table;
```

---

## Recovery Target Options Reference

```ini
# Option 1: Recover to a specific timestamp
recovery_target_time = '2025-06-19 14:30:00 UTC'
# Format: 'YYYY-MM-DD HH:MM:SS timezone'

# Option 2: Recover to just before a specific transaction
recovery_target_xid = '12345678'

# Option 3: Recover to a specific LSN (Log Sequence Number)
recovery_target_lsn = '0/15D5A50'
# Get the LSN of a specific WAL file: SELECT pg_lsn('0/15D5A50');

# Option 4: Recover to a named restore point
recovery_target_name = 'before_migration_20250619'
# (Created with: SELECT pg_create_restore_point('name');)

# Option 5: Recover to a consistent state ASAP (after hardware failure)
recovery_target = 'immediate'

# Include or exclude the target transaction in recovery
recovery_target_inclusive = true    # Default; include the target point

# What happens when recovery reaches the target
recovery_target_action = 'promote'  # Open for writes immediately (default since PG15)
recovery_target_action = 'pause'    # Pause recovery; inspect before promoting
recovery_target_action = 'shutdown' # Shut down; manually start to promote
```

### Using 'pause' to Inspect Before Promoting

```sql
-- Set recovery_target_action = 'pause' in postgresql.auto.conf

-- After recovery pauses, check the database state:
SELECT pg_last_xact_replay_timestamp();  -- What time has been recovered to?
SELECT count(*) FROM orders;             -- Is the data correct?

-- If the data looks good, promote:
SELECT pg_promote();

-- If you need to recover further (went too far), shutdown and adjust target:
-- SHUT DOWN, change recovery_target_time, start again
```

---

## Restore Command Templates

```ini
# Local archive directory
restore_command = 'cp /wal_archive/%f %p'

# Remote server with rsync
restore_command = 'rsync -a backup-server:/wal_archive/%f %p'

# WAL-G
restore_command = 'wal-g wal-fetch %f %p'

# pgBackRest
restore_command = 'pgbackrest --stanza=main archive-get %f %p'

# With error logging
restore_command = 'cp /wal_archive/%f %p 2>>/var/log/postgresql/restore.log'
```

---

## WAL Archiving Monitoring

```sql
-- Check archiving health
SELECT archived_count,
       last_archived_wal,
       last_archived_time,
       failed_count,
       last_failed_wal,
       last_failed_time,
       now() - last_archived_time AS time_since_last_archive
FROM pg_stat_archiver;

-- Check WAL directory size (should not grow unbounded)
-- From the OS:
-- du -sh /var/lib/postgresql/data/pg_wal/
-- ls -1 /var/lib/postgresql/data/pg_wal/ | wc -l

-- Check current WAL position
SELECT pg_current_wal_lsn(),
       pg_walfile_name(pg_current_wal_lsn()) AS current_wal_file;
```

**Troubleshooting archive failures:**
1. Check `last_failed_wal` and `last_failed_time` in `pg_stat_archiver`
2. Run the archive command manually to see error output: `cp /var/lib/postgresql/data/pg_wal/00000001000000000000001 /wal_archive/`
3. Common causes: disk full, permission denied, network timeout, archive server unreachable
4. After fixing: `SELECT pg_switch_wal();` to force archival attempt
