# PostgreSQL Backup Commands Reference

## pg_dump — Logical Backup

### Format Options
| Format | Flag | Description | Restore with |
|--------|------|-------------|--------------|
| Custom | `-Fc` | Compressed binary; selective restore | `pg_restore` |
| Directory | `-Fd` | One file per object; parallel backup/restore | `pg_restore` |
| Plain SQL | `-Fp` | Text SQL; portable; slow restore | `psql` |
| TAR | `-Ft` | Single TAR archive | `pg_restore` |

**Recommendation:** Use `-Fc` for most databases. Use `-Fd -j N` for large databases to enable parallel dump.

### Content Selection
```bash
# Dump specific schema
pg_dump -Fc -n myschema -d mydb -f myschema.dump

# Dump specific tables
pg_dump -Fc -t orders -t order_items -d mydb -f tables.dump

# Exclude tables by name
pg_dump -Fc -T log_events -T audit_log -d mydb -f mydb_no_logs.dump

# Exclude table data but keep schema
pg_dump -Fc --exclude-table-data=log_events -d mydb -f mydb.dump

# Schema only (DDL, no rows)
pg_dump -Fc -s -d mydb -f mydb_schema.dump

# Data only (INSERT statements or COPY)
pg_dump -Fc -a -d mydb -f mydb_data.dump
pg_dump -Fp -a --column-inserts -d mydb -f mydb_inserts.sql  # Row-by-row INSERTs

# Parallel dump (directory format only)
pg_dump -Fd -j 4 -d mydb -f /backup/mydb_dir/
```

### Connection Options
```bash
# Explicit connection parameters
pg_dump -h hostname -p 5432 -U username -d dbname -Fc -f backup.dump

# Using a connection string
pg_dump "postgresql://user:pass@host:5432/dbname" -Fc -f backup.dump
pg_dump "service=mydb" -Fc -f backup.dump  # Using pg_service.conf

# Using .pgpass for password (preferred over PGPASSWORD)
# ~/.pgpass: hostname:port:database:username:password
# chmod 600 ~/.pgpass
```

### Handling Long-Running Dumps
```bash
# Set a lock wait timeout to avoid hanging on locked tables
pg_dump -Fc \
    --lock-wait-timeout=30000 \   # 30 seconds max wait for table lock
    -d mydb \
    -f mydb_backup.dump

# Compress on the fly (alternative to -Fc compression)
pg_dump -Fp -d mydb | gzip -9 > mydb_backup.sql.gz
```

---

## pg_restore — Restore from Custom/Directory/TAR Format

### Basic Restore Operations
```bash
# Restore to existing empty database
pg_restore -d mydb mydb_backup.dump

# Restore with parallel jobs
pg_restore -d mydb -j 4 -v mydb_backup.dump

# Create target database then restore
pg_restore -C -d postgres mydb_backup.dump   # -C creates db named in dump

# Restore to different database name
createdb newdb
pg_restore -d newdb mydb_backup.dump
```

### Selective Restore
```bash
# List dump contents
pg_restore -l mydb_backup.dump

# Save and edit the list, then restore selected objects
pg_restore -l mydb_backup.dump > restore_list.txt
# Edit restore_list.txt: comment out objects you don't want with semicolons
pg_restore -d mydb -L restore_list.txt mydb_backup.dump

# Restore only one schema
pg_restore -d mydb -n myschema mydb_backup.dump

# Restore only one table
pg_restore -d mydb -t orders mydb_backup.dump

# Restore schema only
pg_restore -d mydb -s mydb_backup.dump

# Restore data only
pg_restore -d mydb -a mydb_backup.dump

# Restore specific section
pg_restore -d mydb --section=pre-data mydb_backup.dump   # DDL
pg_restore -d mydb --section=data mydb_backup.dump       # Data
pg_restore -d mydb --section=post-data mydb_backup.dump  # Indexes, constraints
```

### Handling Restore Errors
```bash
# Ignore errors (useful for restoring into a database with pre-existing objects)
pg_restore -d mydb --if-exists -c mydb_backup.dump   # Drop and recreate objects
pg_restore -d mydb --no-owner mydb_backup.dump        # Don't set object ownership

# Disable triggers during data restore (for FK constraint violations)
pg_restore -d mydb --disable-triggers mydb_backup.dump  # Requires superuser

# Restore with single-transaction (all or nothing)
pg_restore -d mydb --single-transaction mydb_backup.dump
```

---

## pg_dumpall — Full Cluster Backup

```bash
# Dump entire cluster (all databases + global objects)
pg_dumpall -f cluster_full.sql

# Globals only (roles, tablespace definitions — no database data)
pg_dumpall --globals-only -f globals.sql

# Schema only across all databases
pg_dumpall --schema-only -f cluster_schema.sql

# Restore cluster backup
psql -f cluster_full.sql postgres

# Restore globals only
psql -f globals.sql postgres
```

**Note:** `pg_dumpall` output is always plain SQL. Use it to capture roles and tablespaces, then use `pg_dump -Fc` for each database for efficient backups.

---

## pg_basebackup — Physical Backup

### Common Invocations
```bash
# Production physical backup (tar + gzip, WAL via streaming)
pg_basebackup \
    -h localhost \
    -U replicator \
    -D /backup/base_$(date +%Y%m%d_%H%M%S) \
    -Ft -z \
    -Xs \
    -P \
    --checkpoint=fast \
    --label="nightly_$(date +%Y%m%d)"

# Plain files with standby configuration (-R flag)
pg_basebackup \
    -h primary.example.com \
    -U replicator \
    -D /var/lib/postgresql/data \
    -Fp -Xs -P -R

# Backup to a specific directory with tablespace remapping
pg_basebackup \
    -h localhost -U replicator \
    -D /backup/base_20250619 \
    -Fp -Xs -P \
    --tablespace-map=/original/tblspc=/backup/tblspc
```

### Parameter Reference
| Parameter | Description |
|-----------|-------------|
| `-D dir` | Target directory (must not exist or be empty) |
| `-Ft` | Tar format (`base.tar.gz` + `pg_wal.tar.gz`) |
| `-Fp` | Plain files (directory mirror) |
| `-z` | gzip compression (only with `-Ft`) |
| `-Z N` | Compression level 1–9 |
| `-Xs` | Include needed WAL by streaming during backup |
| `-Xf` | Include needed WAL by fetching after backup completes |
| `-Xn` | No WAL (use only if WAL archiving is separately enabled) |
| `-P` | Show progress |
| `-R` | Write `standby.signal` + `primary_conninfo` for standby |
| `--checkpoint=fast` | Trigger immediate checkpoint; faster backup start |
| `--label=string` | Backup label string |
| `--tablespace-map=old=new` | Remap tablespace paths |

### Required Prerequisites
```sql
-- Role must have REPLICATION privilege
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'strong_password';

-- Check pg_hba.conf has entry for replication connections:
-- host  replication  replicator  10.0.0.0/8  scram-sha-256

-- postgresql.conf must have:
-- wal_level = replica (or higher)
-- max_wal_senders >= 2
```

---

## WAL-G Quick Reference (Production Backup Tool)

```bash
# Environment variables for S3
export WALG_S3_PREFIX="s3://my-bucket/postgresql"
export AWS_ACCESS_KEY_ID="..."
export AWS_SECRET_ACCESS_KEY="..."

# Push a base backup
wal-g backup-push /var/lib/postgresql/data

# List available backups
wal-g backup-list

# Restore latest backup
wal-g backup-fetch /var/lib/postgresql/data LATEST

# Restore specific backup
wal-g backup-fetch /var/lib/postgresql/data base_000000010000000000000002

# WAL archiving commands (used in postgresql.conf)
# archive_command = 'wal-g wal-push %p'
# restore_command = 'wal-g wal-fetch %f %p'
```

---

## pgBackRest Quick Reference (Enterprise Backup Tool)

```bash
# Initialize (run once)
pgbackrest --stanza=main stanza-create

# Full backup
pgbackrest --stanza=main --type=full backup

# Differential backup (since last full)
pgbackrest --stanza=main --type=diff backup

# Incremental backup (since last any)
pgbackrest --stanza=main --type=incr backup

# List backups
pgbackrest --stanza=main info

# Restore latest
pgbackrest --stanza=main restore

# PITR restore to specific time
pgbackrest --stanza=main \
    --type=time \
    --target="2025-06-19 14:30:00" \
    restore

# Verify backup integrity
pgbackrest --stanza=main check
```

### pgBackRest Configuration (/etc/pgbackrest/pgbackrest.conf)
```ini
[global]
repo1-path=/backup/pgbackrest
repo1-retention-full=2
repo1-retention-diff=7
# For S3:
# repo1-type=s3
# repo1-s3-bucket=my-bucket
# repo1-s3-region=us-east-1

[global:archive-push]
compress-level=3

[main]
pg1-path=/var/lib/postgresql/data
```
