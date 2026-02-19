# PostgreSQL Recovery Scenarios

## Scenario 1: Accidental Table DROP

**Situation:** A developer ran `DROP TABLE orders;` on production at 15:45 UTC.
WAL archiving is enabled with `archive_timeout = 300`.
Last base backup: today at 02:00 UTC.

**Recovery target:** Time just before the DROP (15:44:55 UTC).

```bash
# Step 1: Verify the drop time from PostgreSQL logs
grep "DROP TABLE" /var/log/postgresql/postgresql-2025-06-19.log
# LOG:  statement: DROP TABLE orders;  (at 15:45:02)

# Step 2: Stop PostgreSQL immediately to prevent further changes
systemctl stop postgresql

# Step 3: Preserve current data directory
mv /var/lib/postgresql/data /var/lib/postgresql/data.after_drop

# Step 4: Restore base backup
mkdir -p /var/lib/postgresql/data
tar -xzf /backup/base_20250619_020000/base.tar.gz -C /var/lib/postgresql/data
chown -R postgres:postgres /var/lib/postgresql/data
chmod 700 /var/lib/postgresql/data

# Step 5: Set recovery target to just before the drop
cat >> /var/lib/postgresql/data/postgresql.auto.conf << 'EOF'
restore_command = 'cp /wal_archive/%f %p'
recovery_target_time = '2025-06-19 15:44:55 UTC'
recovery_target_action = 'pause'
recovery_target_inclusive = true
EOF

touch /var/lib/postgresql/data/recovery.signal

# Step 6: Start recovery
systemctl start postgresql

# Step 7: Wait for recovery to pause, then verify
psql -c "SELECT count(*) FROM orders;"
# Expected: table exists with expected row count
psql -c "SELECT pg_last_xact_replay_timestamp();"
# Should show approximately 15:44:55

# Step 8: If data is correct, promote to primary
psql -c "SELECT pg_promote();"
```

---

## Scenario 2: Hardware Failure with No Data Loss Goal

**Situation:** The primary database server's OS disk failed. A physical standby exists with streaming replication. Standby is 30 seconds behind.

**Decision:** Fail over to the standby.

```bash
# On the STANDBY server:

# Step 1: Verify the standby is not already primary
psql -c "SELECT pg_is_in_recovery();"
# Should return: t (true = this is a standby)

# Step 2: Check lag before promoting
psql -c "
SELECT pg_last_wal_receive_lsn() AS received,
       pg_last_wal_replay_lsn() AS replayed,
       now() - pg_last_xact_replay_timestamp() AS lag;
"

# Step 3: Promote the standby
# Method A: pg_ctl (from OS)
pg_ctl promote -D /var/lib/postgresql/data

# Method B: SQL (PostgreSQL 12+)
psql -c "SELECT pg_promote(wait => true, wait_seconds => 60);"

# Step 4: Verify promotion succeeded
psql -c "SELECT pg_is_in_recovery();"
# Should return: f (false = this is now a primary)

# Step 5: Update application connection strings to point to new primary
# (Update DNS, load balancer VIP, or connection string configuration)

# Step 6: Verify application connectivity
psql -h new-primary.example.com -d mydb -c "SELECT count(*) FROM orders;"
```

**Post-failover cleanup — Rebuild old primary as standby:**
```bash
# When old primary hardware is repaired:

# Method 1: Fresh basebackup (simplest, always safe)
pg_basebackup \
    -h new-primary.example.com \
    -U replicator \
    -D /var/lib/postgresql/data \
    -Fp -Xs -P -R
systemctl start postgresql

# Method 2: pg_rewind (faster, reuses unchanged files)
# Requires: wal_log_hints = on OR data checksums enabled on old primary
pg_rewind \
    --target-pgdata=/var/lib/postgresql/data \
    --source-server="host=new-primary.example.com user=replicator"
touch /var/lib/postgresql/data/standby.signal
# Update postgresql.auto.conf primary_conninfo to point to new primary
systemctl start postgresql
```

---

## Scenario 3: Corrupt Data File (Partial Failure)

**Situation:** PostgreSQL is running but crashes with `invalid page in block N of relation base/16384/24601` — a single data file is corrupt.

**Goal:** Recover the specific table without a full PITR.

```bash
# Step 1: Identify the corrupted table
# The OID 24601 corresponds to a table; find which table:
psql -c "SELECT relname, oid FROM pg_class WHERE oid = 24601 OR relfilenode = 24601;"

# Step 2: Check if a logical backup is available for this table
pg_restore -l mydb_backup.dump | grep orders
# If the table is in the backup, restore just that table:
pg_restore -d mydb -t orders mydb_backup.dump
# WARNING: this restores data as of the backup time, losing changes since then

# Step 3: If you need current data via PITR
# Set recovery_target = 'immediate' to recover to a consistent state fast
cat >> /var/lib/postgresql/data/postgresql.auto.conf << 'EOF'
restore_command = 'cp /wal_archive/%f %p'
recovery_target = 'immediate'
recovery_target_action = 'promote'
EOF

# Step 4: Use pg_filedump to inspect the corrupt page (advanced)
# pg_filedump -i -f /var/lib/postgresql/data/base/16384/24601
```

---

## Scenario 4: Recovering from a Logical Backup

**Situation:** Need to restore a specific database to a new server for testing or migration. A `pg_dump -Fc` backup exists.

```bash
# Step 1: On the target server, create the target database
psql -c "CREATE DATABASE mydb_restored;"

# Step 2: Restore with parallel jobs
pg_restore \
    -h target-server \
    -d mydb_restored \
    -j 4 \
    -v \
    /backup/mydb_20250619.dump

# Step 3: Verify the restore
psql -h target-server -d mydb_restored -c "
SELECT table_name,
       (xpath('/row/count/text()',
              query_to_xml('SELECT count(*) FROM ' || quote_ident(table_name), true, false, ''))
       )[1]::text::int AS row_count
FROM information_schema.tables
WHERE table_schema = 'public'
ORDER BY table_name;
"

# Alternative quick row count check on specific tables:
psql -h target-server -d mydb_restored -c "SELECT count(*) FROM orders;"
psql -h source-server -d mydb -c "SELECT count(*) FROM orders;"

# Step 4: Restore global objects (roles) if needed
pg_dumpall --globals-only | psql -h target-server postgres
```

---

## Scenario 5: Recovery to a Different Server (Cross-Host PITR)

**Situation:** Restore to a standby server to verify a backup, or recover to a new server after complete primary failure.

```bash
# On the RECOVERY TARGET SERVER:

# Step 1: Install the same PostgreSQL version as the source
# (pg_dump/pg_restore can handle cross-minor-versions)
# For physical backup: must be same major version

# Step 2: Transfer the base backup
rsync -a --progress /backup/base_20250619_020000/ recovery-server:/var/lib/postgresql/data/

# Step 3: Transfer WAL archive (or mount shared storage)
rsync -a --progress /wal_archive/ recovery-server:/wal_archive/

# Step 4: Set up recovery configuration on recovery-server
cat >> /var/lib/postgresql/data/postgresql.auto.conf << 'EOF'
restore_command = 'cp /wal_archive/%f %p'
recovery_target_time = '2025-06-19 14:30:00 UTC'
recovery_target_action = 'promote'
EOF

touch /var/lib/postgresql/data/recovery.signal
chown -R postgres:postgres /var/lib/postgresql/data

# Step 5: Start PostgreSQL on recovery server
systemctl start postgresql

# Step 6: Verify data on recovery server
psql -h recovery-server -d mydb -c "SELECT count(*) FROM orders;"

# Step 7: If satisfied, update DNS/application to point to recovery server
```

---

## Recovery Verification Checklist

After any recovery, verify before declaring success:

```sql
-- 1. Confirm recovery is complete (not in recovery mode)
SELECT pg_is_in_recovery();   -- Must return false

-- 2. Check recovered timestamp
SELECT pg_last_xact_replay_timestamp();  -- Should show expected recovery time

-- 3. Verify critical table row counts
SELECT 'orders', count(*) FROM orders
UNION ALL
SELECT 'customers', count(*) FROM customers
UNION ALL
SELECT 'products', count(*) FROM products;

-- 4. Check for any data inconsistencies (FK violations)
-- Run application smoke tests

-- 5. Check database for corruption using pg_dump test
-- (Takes time but is thorough)
pg_dump -Fc -d mydb -f /dev/null

-- 6. Verify sequences are correct
SELECT sequence_name, last_value
FROM information_schema.sequences
ORDER BY sequence_name;

-- 7. Check extensions are still functional
SELECT extname, extversion FROM pg_extension;
```
