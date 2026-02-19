# PostgreSQL Backup and Recovery Examples

## Scenario 1: Setting Up Nightly Logical Backups

**Situation:** A development team needs automated nightly backups for a 50GB PostgreSQL database. Requirements: simple restore, single-database scope, 30-day retention.

**Backup script (`/usr/local/bin/pg_backup_nightly.sh`):**
```bash
#!/bin/bash
set -e

DB_NAME="myapp"
BACKUP_DIR="/backups/postgresql"
BACKUP_DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${BACKUP_DIR}/${DB_NAME}_${BACKUP_DATE}.dump"
RETENTION_DAYS=30

mkdir -p "${BACKUP_DIR}"

echo "Starting backup of ${DB_NAME} at $(date)"

# Custom format with parallel dump
pg_dump -Fc -j 4 -v \
    -d "${DB_NAME}" \
    --lock-wait-timeout=30000 \
    -f "${BACKUP_FILE}"

echo "Backup completed: ${BACKUP_FILE} ($(du -sh ${BACKUP_FILE} | cut -f1))"

# Verify the backup is readable
pg_restore -l "${BACKUP_FILE}" > /dev/null && echo "Backup verification: OK"

# Delete backups older than 30 days
find "${BACKUP_DIR}" -name "${DB_NAME}_*.dump" -mtime +${RETENTION_DAYS} -delete
echo "Cleaned up backups older than ${RETENTION_DAYS} days"
```

```bash
# Make executable and schedule with cron
chmod +x /usr/local/bin/pg_backup_nightly.sh

# Run at 2 AM daily
echo "0 2 * * * postgres /usr/local/bin/pg_backup_nightly.sh >> /var/log/pg_backup.log 2>&1" \
    | crontab -
```

**Restore from this backup:**
```bash
# Create target database
createdb myapp_restore

# Restore with parallel jobs
pg_restore -d myapp_restore -j 4 -v /backups/postgresql/myapp_20250619_020000.dump

# Verify row counts match
psql -d myapp -c "SELECT count(*) FROM orders;"
psql -d myapp_restore -c "SELECT count(*) FROM orders;"
```

---

## Scenario 2: Physical Backup Setup with WAL Archiving

**Situation:** A production database needs PITR capability. Requirements: RPO < 5 minutes, fast physical restore, point-in-time recovery.

**Step 1 — Configure postgresql.conf (requires restart):**
```bash
cat >> /etc/postgresql/16/main/postgresql.conf << 'EOF'
# WAL archiving for PITR
wal_level = replica
archive_mode = on
archive_command = 'cp %p /wal_archive/%f'
archive_timeout = 300           # Force WAL switch every 5 minutes
max_wal_senders = 5
wal_keep_size = 2048            # 2 GB of WAL retained locally
EOF

systemctl restart postgresql
```

**Step 2 — Verify archiving is working:**
```sql
SELECT pg_switch_wal();  -- Force a WAL switch

SELECT archived_count, last_archived_wal, last_archived_time, failed_count
FROM pg_stat_archiver;
-- expected: archived_count > 0, failed_count = 0
```

**Step 3 — Take an initial base backup:**
```bash
mkdir -p /backup/base
pg_basebackup \
    -h localhost \
    -U replicator \
    -D /backup/base/$(date +%Y%m%d) \
    -Ft -z \
    -Xs \
    -P \
    --checkpoint=fast \
    --label="daily_$(date +%Y%m%d)"

# List contents to verify
tar -tzf /backup/base/20250619/base.tar.gz | head -20
```

**Step 4 — Weekly base backup rotation script:**
```bash
#!/bin/bash
BACKUP_BASE="/backup/base"
DATE=$(date +%Y%m%d_%H%M%S)

# Take new base backup
pg_basebackup \
    -h localhost -U replicator \
    -D "${BACKUP_BASE}/${DATE}" \
    -Ft -z -Xs -P \
    --checkpoint=fast

# Keep only last 4 weekly base backups
ls -dt "${BACKUP_BASE}/"* | tail -n +5 | xargs rm -rf

echo "Base backup completed: ${BACKUP_BASE}/${DATE}"
```

---

## Scenario 3: Configuring WAL-G for Cloud Backup

**Situation:** Production database needs backups to Amazon S3 with encryption for compliance.

**Step 1 — Install WAL-G and configure environment:**
```bash
# /etc/postgresql/wal-g.env
WALG_S3_PREFIX=s3://my-company-backups/postgresql/prod
AWS_REGION=us-east-1
WALG_COMPRESSION_METHOD=brotli
WALG_PGP_KEY_PATH=/etc/postgresql/backup_public.key
WALG_UPLOAD_CONCURRENCY=4
WALG_DOWNLOAD_CONCURRENCY=4

# Retention
WALG_RETAIN_FULL=4          # Keep 4 full backups
WALG_RETAIN_AFTER_FULL=3    # Keep 3 delta backups after each full
```

**Step 2 — postgresql.conf:**
```ini
wal_level = replica
archive_mode = on
archive_command = '. /etc/postgresql/wal-g.env && wal-g wal-push %p'
archive_timeout = 60        # Archive WAL every 60 seconds (1-minute RPO)
```

**Step 3 — Weekly full backup via cron:**
```bash
# /etc/cron.d/wal-g-backup
0 1 * * 0 postgres . /etc/postgresql/wal-g.env && wal-g backup-push /var/lib/postgresql/data >> /var/log/wal-g-backup.log 2>&1

# Verify backups exist
# wal-g backup-list
```

---

## Scenario 4: PITR to a Specific Time

**Situation:** An application bug wrote incorrect prices to the `products` table at 15:30 UTC. The error was discovered at 16:00 UTC. Need to recover to 15:29:00 UTC.

**Step 1 — Identify recovery target and confirm WAL archive is complete:**
```bash
ls /wal_archive/ | sort | tail -5
# Verify WAL segments exist up to 15:30 UTC
```

**Step 2 — Stop PostgreSQL:**
```bash
systemctl stop postgresql
```

**Step 3 — Restore base backup:**
```bash
mv /var/lib/postgresql/data /var/lib/postgresql/data.incorrect
mkdir -p /var/lib/postgresql/data
chmod 700 /var/lib/postgresql/data

# Find most recent base backup before 15:30
ls -la /backup/base/
# Use 20250619_020000 (backup from 02:00 today)

tar -xzf /backup/base/20250619_020000/base.tar.gz -C /var/lib/postgresql/data
chown -R postgres:postgres /var/lib/postgresql/data
```

**Step 4 — Configure PITR target:**
```bash
cat >> /var/lib/postgresql/data/postgresql.auto.conf << 'EOF'
restore_command = 'cp /wal_archive/%f %p'
recovery_target_time = '2025-06-19 15:29:00 UTC'
recovery_target_action = 'pause'
recovery_target_inclusive = true
EOF

touch /var/lib/postgresql/data/recovery.signal
```

**Step 5 — Start and verify:**
```bash
systemctl start postgresql

# Wait for recovery to pause, then check
psql -c "SELECT pg_last_xact_replay_timestamp();"
# Should show: 2025-06-19 15:28:58+00

psql -c "SELECT id, name, price FROM products WHERE id IN (1, 2, 3);"
# Verify prices look correct (before the bug)

psql -c "SELECT count(*) FROM orders WHERE created_at > '2025-06-19 15:00:00';"
# Note: some orders placed between 15:00 and 15:30 may need to be re-entered
```

**Step 6 — Promote if data is correct:**
```bash
psql -c "SELECT pg_promote();"
# or: pg_ctl promote -D /var/lib/postgresql/data

# Verify database is now writable
psql -c "SELECT pg_is_in_recovery();"
# Should return: f
```

**Step 7 — Re-apply lost transactions if needed:**
If orders were placed between 15:29 and 15:30, they are lost. Recover them from:
- Application logs
- The preserved `.incorrect` data directory:
  ```bash
  # Start the corrupt data directory on a different port to extract data
  pg_ctl start -D /var/lib/postgresql/data.incorrect -o "-p 5433"
  psql -p 5433 -d mydb -c "SELECT * FROM orders WHERE created_at BETWEEN '2025-06-19 15:29' AND '2025-06-19 15:30:30';"
  # Re-insert these rows into the recovered database
  ```

---

## Scenario 5: Cross-Version Migration Using pg_dump

**Situation:** Need to upgrade from PostgreSQL 14 to PostgreSQL 17 with minimal downtime.

**Strategy:** Dump from old, restore to new, cut over application traffic.

**Step 1 — On old PostgreSQL 14 server:**
```bash
# Pre-migration: check for incompatibilities
/usr/lib/postgresql/17/bin/pg_dump --version
# pg_dump (PostgreSQL) 17.x

# Use version-17 pg_dump against version-14 source (backward compatible)
/usr/lib/postgresql/17/bin/pg_dump \
    -h pg14-server \
    -U myapp \
    -d myapp \
    -Fc -j 4 -v \
    -f /migration/myapp_pg14_$(date +%Y%m%d_%H%M%S).dump
```

**Step 2 — Restore to PostgreSQL 17:**
```bash
# On PG17 server, create database and roles
psql -c "CREATE USER myapp WITH PASSWORD 'password';"
psql -c "CREATE DATABASE myapp OWNER myapp;"

# Restore
/usr/lib/postgresql/17/bin/pg_restore \
    -h pg17-server \
    -d myapp \
    -j 4 -v \
    /migration/myapp_pg14_20250619_020000.dump

# Update statistics after migration
/usr/lib/postgresql/17/bin/vacuumdb \
    -h pg17-server \
    -d myapp \
    --analyze-in-stages \
    -v
```

**Step 3 — Verify before cutover:**
```bash
# Compare row counts on both servers
for table in orders customers products; do
    old=$(psql -h pg14-server -d myapp -t -c "SELECT count(*) FROM $table;")
    new=$(psql -h pg17-server -d myapp -t -c "SELECT count(*) FROM $table;")
    echo "$table: PG14=$old PG17=$new"
done

# Run application integration tests against PG17
```

**Step 4 — Cutover:**
```bash
# 1. Put application in maintenance mode (no new writes)
# 2. Take final delta dump of rows changed since initial dump
psql -h pg14-server -d myapp -c "
SELECT * FROM orders WHERE updated_at > '2025-06-19 02:00:00'
" | psql -h pg17-server -d myapp -c "COPY orders FROM STDIN;"
# (or use a more precise CDC approach)

# 3. Update application connection string to pg17-server
# 4. Remove maintenance mode
```
