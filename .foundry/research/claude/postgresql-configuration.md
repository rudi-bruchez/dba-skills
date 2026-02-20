# PostgreSQL Configuration & Maintenance

## Overview
PostgreSQL configuration is managed primarily through `postgresql.conf` (and `postgresql.auto.conf` for `ALTER SYSTEM` changes). Some parameters require a server restart, others just a reload, and some can be set per-session. Autovacuum, extension management, and `pg_upgrade` are critical maintenance operations.

---

## Configuration Files

### File Hierarchy
| File | Purpose | How to change |
|------|---------|---------------|
| `postgresql.conf` | Main configuration | Edit + reload (or restart) |
| `postgresql.auto.conf` | Override via `ALTER SYSTEM` | Never edit manually |
| `pg_hba.conf` | Authentication rules | Edit + reload |
| `pg_ident.conf` | Username mapping | Edit + reload |

### Locating Configuration Files
```sql
SHOW config_file;         -- Path to postgresql.conf
SHOW hba_file;            -- Path to pg_hba.conf
SHOW data_directory;      -- Data directory path
```

### Viewing Current Settings
```sql
-- All current settings with source
SELECT name, setting, unit, category, short_desc, context, source, sourcefile, sourceline
FROM pg_settings
ORDER BY category, name;

-- Find non-default settings
SELECT name, setting, unit, source, sourcefile
FROM pg_settings
WHERE source NOT IN ('default', 'override')
ORDER BY name;

-- Settings that require restart
SELECT name, setting, pending_restart
FROM pg_settings
WHERE pending_restart = true;

-- Show a specific setting
SHOW shared_buffers;
SHOW ALL;
```

### Applying Configuration Changes
```sql
-- Reload without restart (for parameters that allow it)
SELECT pg_reload_conf();

-- Reload from OS
-- systemctl reload postgresql-16
-- pg_ctl reload -D /var/lib/postgresql/data

-- Check if restart is needed
SELECT name, setting, pending_restart
FROM pg_settings
WHERE pending_restart = true;
```

### ALTER SYSTEM (writes to postgresql.auto.conf)
```sql
ALTER SYSTEM SET shared_buffers = '4GB';
ALTER SYSTEM SET work_mem = '64MB';
ALTER SYSTEM RESET work_mem;        -- Remove from auto.conf
ALTER SYSTEM RESET ALL;             -- Clear all auto.conf settings
SELECT pg_reload_conf();            -- Apply reload-able changes
```

---

## Critical Configuration Parameters

### Memory
```ini
# Size of shared buffer pool (PostgreSQL's main cache)
# Rule of thumb: 25% of RAM, but 8GB is often sufficient even on large servers
shared_buffers = 4GB

# Per-sort/per-hash-table memory allocation
# Multiply by max_connections * parallel_workers for total potential usage
# Set conservatively for OLTP; higher for analytics
work_mem = 64MB

# Maintenance operations: VACUUM, CREATE INDEX, pg_dump
maintenance_work_mem = 512MB

# Planner's estimate of total RAM available for caching
# Set to 50-75% of RAM; does NOT actually allocate memory
effective_cache_size = 12GB

# WAL buffers in shared memory
wal_buffers = 64MB    # Default -1 = 1/32 of shared_buffers, max 64MB; manual override
```

### Connections
```ini
max_connections = 200               # Each connection uses ~5-10MB
superuser_reserved_connections = 3  # Always reserve for emergency access
```

### WAL & Checkpoints
```ini
# WAL durability and content
wal_level = replica                 # 'minimal', 'replica', 'logical'
fsync = on                          # Never disable in production!
synchronous_commit = on             # 'off' for async (faster, small data loss risk)
full_page_writes = on               # Required for crash safety (keep on)
wal_compression = on                # Compress full-page writes (small CPU overhead)

# Checkpoint frequency and spread
checkpoint_completion_target = 0.9  # Spread I/O over 90% of checkpoint interval
checkpoint_timeout = 5min           # Max time between checkpoints
max_wal_size = 2GB                  # Target WAL size before checkpoint (PG10+)
min_wal_size = 80MB                 # Don't shrink WAL below this

# WAL archiving
archive_mode = on
archive_command = 'cp %p /wal_archive/%f'
archive_timeout = 300               # Force WAL switch every 5 minutes
```

### Query Planning
```ini
# Cost constants — tune for hardware type
random_page_cost = 1.1             # SSD: 1.1; spinning disk: 4.0 (default)
seq_page_cost = 1.0                # Baseline cost (rarely changed)
cpu_tuple_cost = 0.01
cpu_index_tuple_cost = 0.005
cpu_operator_cost = 0.0025
effective_io_concurrency = 200     # SSD: 100-200; HDD: 2; RAID: 300+

# Statistics
default_statistics_target = 100   # Increase to 200-500 for complex queries

# Parallel query
max_worker_processes = 8
max_parallel_workers = 8
max_parallel_workers_per_gather = 4
max_parallel_maintenance_workers = 4
parallel_leader_participation = on
min_parallel_table_scan_size = 8MB
min_parallel_index_scan_size = 512kB

# JIT compilation (PG11+)
jit = on                           # Disable for OLTP (overhead not worth it for short queries)
jit_above_cost = 100000            # Only JIT-compile queries more expensive than this
```

### Logging
```ini
log_destination = 'stderr'         # 'stderr', 'csvlog', 'jsonlog' (PG17+), 'syslog'
logging_collector = on             # Required for csvlog/jsonlog
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d.log'
log_rotation_age = 1d
log_rotation_size = 100MB
log_truncate_on_rotation = on

# What to log
log_min_messages = warning
log_min_error_statement = error
log_min_duration_statement = 1000  # Log queries taking > 1000ms (1 second)
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
log_temp_files = 64                # Log temp files larger than 64kB
log_autovacuum_min_duration = 250  # Log autovacuums taking > 250ms

# Log format
log_line_prefix = '%m [%p] %q%u@%d '  # timestamp [pid] user@db
log_statement = 'ddl'              # Log DDL statements; 'mod' or 'all' for more

# Auto-explain (extension)
# shared_preload_libraries = 'auto_explain'
auto_explain.log_min_duration = 5000   # Explain queries > 5 seconds
auto_explain.log_analyze = on
auto_explain.log_buffers = on
auto_explain.log_nested_statements = on
```

### Lock Management
```ini
deadlock_timeout = 1s              # Time to wait before checking for deadlock
lock_timeout = 0                   # 0 = no timeout; set in app code
statement_timeout = 0              # 0 = no timeout; set per role
idle_in_transaction_session_timeout = 300000  # Kill idle-in-txn after 5 min (ms)
idle_session_timeout = 0           # Kill fully idle sessions (PG14+)
```

---

## Autovacuum Configuration

### How Autovacuum Works
- Background worker daemon checks tables periodically
- Triggers vacuum when dead tuples exceed threshold: `autovacuum_vacuum_threshold + autovacuum_vacuum_scale_factor * n_live_tup`
- Triggers analyze when changed tuples exceed: `autovacuum_analyze_threshold + autovacuum_analyze_scale_factor * n_live_tup`
- Uses cost-based throttling to limit I/O impact

### Global Autovacuum Parameters
```ini
# Enable/disable autovacuum
autovacuum = on                              # Never disable in production

# Worker processes
autovacuum_max_workers = 3                   # Number of parallel workers
autovacuum_naptime = 1min                    # How often to check each database

# Triggering thresholds
autovacuum_vacuum_threshold = 50             # Min dead tuples to trigger vacuum
autovacuum_vacuum_scale_factor = 0.20        # + 20% of live tuples
autovacuum_analyze_threshold = 50            # Min changes to trigger analyze
autovacuum_analyze_scale_factor = 0.10       # + 10% of live tuples

# For large tables (millions of rows), scale factors produce huge thresholds.
# Override per-table for better control.

# Freeze settings
autovacuum_freeze_max_age = 200000000        # Freeze at 200M XID age
autovacuum_multixact_freeze_max_age = 400000000

# Cost-based throttling (limits I/O impact)
autovacuum_vacuum_cost_delay = 2ms           # Pause duration per cost limit
autovacuum_vacuum_cost_limit = -1            # -1 = use vacuum_cost_limit
vacuum_cost_limit = 200                      # Cost units before pausing
vacuum_cost_delay = 0                        # No delay for manual vacuum
```

### Per-Table Autovacuum Override
```sql
-- High-churn table: vacuum more aggressively
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.02,    -- 2% dead tuples (default 20%)
    autovacuum_analyze_scale_factor = 0.01,   -- 1% changes for analyze
    autovacuum_vacuum_cost_delay = 2,          -- Faster I/O (ms)
    autovacuum_vacuum_cost_limit = 800         -- More work per cycle
);

-- Audit/log table (append-only): disable unnecessary vacuuming
ALTER TABLE audit_log SET (
    autovacuum_enabled = false
);
-- Or just increase thresholds
ALTER TABLE audit_log SET (
    autovacuum_vacuum_scale_factor = 0.5
);

-- Tables approaching XID wraparound: aggressive freeze
ALTER TABLE critical_table SET (
    autovacuum_freeze_max_age = 50000000,
    autovacuum_vacuum_freeze_min_age = 0
);
```

### Monitoring Autovacuum Activity
```sql
-- Currently running autovacuums
SELECT pid,
       datname,
       relid::regclass AS table_name,
       phase,
       heap_blks_total,
       heap_blks_vacuumed,
       round(100.0 * heap_blks_vacuumed / NULLIF(heap_blks_total, 0), 1) AS pct_done,
       num_dead_tuples,
       autovacuum_count
FROM pg_stat_progress_vacuum
JOIN pg_stat_activity USING (pid);

-- Tables most in need of vacuum
SELECT schemaname,
       relname,
       n_dead_tup,
       n_live_tup,
       round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 1) AS dead_pct,
       last_autovacuum,
       last_autoanalyze,
       autovacuum_count,
       autoanalyze_count
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY dead_pct DESC, n_dead_tup DESC
LIMIT 20;

-- Tables autovacuum is struggling with (high dead tuple rate)
SELECT relname,
       n_dead_tup,
       autovacuum_count,
       round(n_dead_tup::numeric / NULLIF(autovacuum_count, 0), 0) AS avg_dead_per_vacuum,
       last_autovacuum,
       now() - last_autovacuum AS time_since_last_vacuum
FROM pg_stat_user_tables
WHERE autovacuum_count > 0
ORDER BY avg_dead_per_vacuum DESC NULLS LAST
LIMIT 10;
```

---

## Extension Management

### Essential Extensions
| Extension | Purpose | Load method |
|-----------|---------|-------------|
| `pg_stat_statements` | Query performance tracking | shared_preload_libraries |
| `auto_explain` | Log slow query plans automatically | shared_preload_libraries |
| `pgaudit` | SQL audit logging | shared_preload_libraries |
| `pg_partman` | Partition management automation | CREATE EXTENSION |
| `postgis` | Geospatial types and functions | CREATE EXTENSION |
| `pg_trgm` | Trigram similarity for LIKE optimization | CREATE EXTENSION |
| `uuid-ossp` | UUID generation functions | CREATE EXTENSION |
| `pgcrypto` | Cryptographic functions | CREATE EXTENSION |
| `tablefunc` | Crosstab and other table functions | CREATE EXTENSION |
| `pg_prewarm` | Preload tables into buffer cache | CREATE EXTENSION |
| `amcheck` | B-tree and heap page verification | CREATE EXTENSION |
| `pageinspect` | Low-level page inspection | CREATE EXTENSION |
| `pgstattuple` | Tuple-level statistics | CREATE EXTENSION |
| `pg_buffercache` | Inspect shared buffer contents | CREATE EXTENSION |
| `pg_visibility` | Visibility map inspection | CREATE EXTENSION |
| `timescaledb` | Time-series optimization | shared_preload_libraries |

### Installing & Managing Extensions
```sql
-- List available extensions
SELECT * FROM pg_available_extensions ORDER BY name;

-- List installed extensions
SELECT extname, extversion, nspname AS schema
FROM pg_extension
JOIN pg_namespace ON extnamespace = pg_namespace.oid
ORDER BY extname;

-- Install extension
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
CREATE EXTENSION pg_trgm SCHEMA pg_catalog;   -- Install in specific schema

-- Upgrade extension
ALTER EXTENSION pg_stat_statements UPDATE;
ALTER EXTENSION pg_stat_statements UPDATE TO '1.10';

-- Drop extension
DROP EXTENSION pg_trgm;
DROP EXTENSION pg_trgm CASCADE;   -- Also drop dependent objects

-- Extensions requiring preload (must restart after adding)
-- shared_preload_libraries = 'pg_stat_statements, auto_explain, pgaudit, timescaledb'
```

### pg_stat_statements Setup
```ini
# postgresql.conf
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.max = 10000        # Track up to 10000 query texts
pg_stat_statements.track = all        # 'top' (default), 'all', 'none'
pg_stat_statements.track_utility = on # Track COPY, VACUUM, etc.
pg_stat_statements.save = on          # Persist across restarts
```

```sql
-- Create the extension in each database you want to track
CREATE EXTENSION pg_stat_statements;

-- Reset statistics
SELECT pg_stat_statements_reset();

-- Top queries by mean execution time
SELECT queryid,
       calls,
       round(mean_exec_time::numeric, 2) AS avg_ms,
       round(total_exec_time::numeric / 1000, 2) AS total_sec,
       rows / NULLIF(calls, 0) AS avg_rows,
       left(query, 80) AS query
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;
```

---

## pg_upgrade — Major Version Upgrade

### Pre-Upgrade Checklist
```bash
# 1. Check for incompatibilities
/usr/lib/postgresql/17/bin/pg_upgrade \
    --old-datadir /var/lib/postgresql/16/main \
    --new-datadir /var/lib/postgresql/17/main \
    --old-bindir /usr/lib/postgresql/16/bin \
    --new-bindir /usr/lib/postgresql/17/bin \
    --check   # Dry run — check only, no changes

# 2. Check extension availability in new version
# 3. Verify pg_hba.conf and postgresql.conf are compatible
# 4. Create a full backup before upgrade
```

### Running pg_upgrade
```bash
# Stop old cluster
pg_ctlcluster 16 main stop

# Run upgrade (--link uses hard links instead of copying — much faster)
/usr/lib/postgresql/17/bin/pg_upgrade \
    --old-datadir /var/lib/postgresql/16/main \
    --new-datadir /var/lib/postgresql/17/main \
    --old-bindir /usr/lib/postgresql/16/bin \
    --new-bindir /usr/lib/postgresql/17/bin \
    --link \
    --jobs 4   # Parallel file copying

# Start new cluster
pg_ctlcluster 17 main start

# Run post-upgrade analyze
/usr/lib/postgresql/17/bin/vacuumdb --all --analyze-in-stages
```

### Post-Upgrade Steps
```sql
-- Run the statistics script generated by pg_upgrade
-- Usually: ./analyze_new_cluster.sh

-- Update extensions
\c mydb
ALTER EXTENSION postgis UPDATE;
ALTER EXTENSION pg_stat_statements UPDATE;

-- Delete old cluster (after verifying new one works)
-- ./delete_old_cluster.sh
```

---

## Tablespace Management

```sql
-- Create tablespace (directory must exist and be owned by postgres)
CREATE TABLESPACE fast_ssd LOCATION '/mnt/nvme/postgresql';

-- Create table in specific tablespace
CREATE TABLE big_table (...) TABLESPACE fast_ssd;

-- Move table to different tablespace (locks table)
ALTER TABLE big_table SET TABLESPACE fast_ssd;

-- Move all tables in database to tablespace
-- (Run for each table individually for production control)

-- View tablespaces
SELECT spcname,
       pg_tablespace_location(oid) AS location,
       pg_size_pretty(pg_tablespace_size(oid)) AS size
FROM pg_tablespace;

-- View tables in each tablespace
SELECT tablename, tablespace
FROM pg_tables
WHERE tablespace IS NOT NULL
ORDER BY tablespace, tablename;
```

---

## Maintenance Operations Schedule

### Daily
```sql
-- Check for bloated tables
SELECT relname, n_dead_tup, last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 100000
ORDER BY n_dead_tup DESC;

-- Check connection usage
SELECT count(*), state FROM pg_stat_activity GROUP BY state;

-- Check for long-running queries
SELECT pid, now() - query_start AS duration, state, query
FROM pg_stat_activity
WHERE now() - query_start > interval '1 hour'
  AND state != 'idle';
```

### Weekly
```sql
-- Analyze statistics freshness
SELECT schemaname, relname, last_analyze, last_autoanalyze,
       n_mod_since_analyze
FROM pg_stat_user_tables
WHERE last_autoanalyze < now() - interval '7 days'
  OR last_autoanalyze IS NULL
ORDER BY n_mod_since_analyze DESC;

-- Check XID age
SELECT datname, age(datfrozenxid) AS xid_age
FROM pg_database
ORDER BY age(datfrozenxid) DESC;

-- Review unused indexes
SELECT indexname, idx_scan, pg_size_pretty(pg_relation_size(indexrelid))
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;
```

### Monthly
```sql
-- Review pg_stat_statements for new problem queries
-- Review role/privilege assignments
-- Test backup restore
-- Review extension versions and update if needed
-- Check for table bloat requiring VACUUM FULL
-- Review partition management (create future partitions)
```

---

## Key pg_settings Context Values

| Context | What it means | How to apply |
|---------|--------------|--------------|
| `internal` | Hard-coded, cannot change | N/A |
| `postmaster` | Server startup only | Restart |
| `sighup` | Reload without restart | `pg_reload_conf()` |
| `superuser` | Superuser can set per-session | `SET parameter = value` |
| `user` | Any user can set per-session | `SET parameter = value` |
| `backend` | Per-connection at connect time | Connection string parameter |

```sql
-- Find which parameters need restart vs reload
SELECT name, context, setting
FROM pg_settings
WHERE context = 'postmaster'
  AND source != 'default'
ORDER BY name;
```

---

## Configuration Tuning Templates

### OLTP Server (8 CPU, 32GB RAM, SSD)
```ini
shared_buffers = 8GB
work_mem = 32MB
maintenance_work_mem = 1GB
effective_cache_size = 24GB
max_connections = 200
random_page_cost = 1.1
effective_io_concurrency = 200
max_worker_processes = 8
max_parallel_workers = 4
max_parallel_workers_per_gather = 2
wal_buffers = 64MB
checkpoint_completion_target = 0.9
max_wal_size = 4GB
min_wal_size = 1GB
jit = off
```

### Analytics/OLAP Server (32 CPU, 128GB RAM, NVMe)
```ini
shared_buffers = 32GB
work_mem = 1GB
maintenance_work_mem = 4GB
effective_cache_size = 96GB
max_connections = 50
random_page_cost = 1.1
effective_io_concurrency = 300
max_worker_processes = 32
max_parallel_workers = 32
max_parallel_workers_per_gather = 16
max_parallel_maintenance_workers = 8
wal_buffers = 64MB
checkpoint_completion_target = 0.9
max_wal_size = 8GB
jit = on
jit_above_cost = 50000
default_statistics_target = 500
enable_partitionwise_join = on
enable_partitionwise_aggregate = on
```

---

## Useful Diagnostic Queries for Configuration

```sql
-- Show settings that differ from defaults
SELECT name, setting, unit, source
FROM pg_settings
WHERE source = 'configuration file'
ORDER BY name;

-- Show all pending restart settings
SELECT name, setting, pending_restart
FROM pg_settings
WHERE pending_restart = true;

-- Shared buffer usage
SELECT count(*) AS total_buffers,
       count(*) FILTER (WHERE isdirty) AS dirty_buffers,
       count(*) FILTER (WHERE usagecount > 0) AS used_buffers
FROM pg_buffercache;

-- Top relations in buffer cache
SELECT c.relname,
       count(*) AS buffer_count,
       pg_size_pretty(count(*) * 8192) AS buffer_size,
       round(100.0 * count(*) / (SELECT count(*) FROM pg_buffercache), 2) AS pct_of_cache
FROM pg_buffercache b
JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
WHERE b.reldatabase IN (0, (SELECT oid FROM pg_database WHERE datname = current_database()))
GROUP BY c.relname
ORDER BY buffer_count DESC
LIMIT 20;
```
