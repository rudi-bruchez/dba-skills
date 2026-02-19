# Key PostgreSQL Statistics Views Reference

## Activity Views

### pg_stat_activity
One row per server process. Most important view for real-time diagnostics.

| Column | Type | Description |
|--------|------|-------------|
| `pid` | int | Process ID |
| `datname` | name | Database name |
| `usename` | name | User name |
| `application_name` | text | Application identifier |
| `client_addr` | inet | Client IP address (null for Unix socket) |
| `state` | text | `active`, `idle`, `idle in transaction`, `idle in transaction (aborted)`, `fastpath function call`, `disabled` |
| `wait_event_type` | text | Category of wait: `Lock`, `LWLock`, `IO`, `Client`, `Activity`, `Extension`, `IPC`, `Timeout`, `BufferPin` |
| `wait_event` | text | Specific wait event name |
| `query` | text | Current or most recent query text |
| `query_start` | timestamptz | When current query started |
| `state_change` | timestamptz | When `state` last changed |
| `backend_type` | text | `client backend`, `autovacuum worker`, `walsender`, `walreceiver`, `background worker`, etc. |

**Key wait events to know:**
- `wait_event_type = 'Lock'` — waiting for a row/table/advisory lock
- `wait_event_type = 'IO'` — waiting for disk I/O (check `wait_event` for specifics)
- `wait_event_type = 'Client'` — waiting for the client to send data
- `wait_event = 'ClientRead'` — idle connection waiting for client query

### pg_stat_database
One row per database. Contains cumulative statistics since last reset.

| Column | Description |
|--------|-------------|
| `datname` | Database name |
| `blks_hit` | Buffer cache hits |
| `blks_read` | Blocks read from disk |
| `xact_commit` | Transactions committed |
| `xact_rollback` | Transactions rolled back |
| `deadlocks` | Deadlocks detected |
| `temp_files` | Temp files created by queries |
| `temp_bytes` | Bytes written to temp files |
| `tup_fetched` | Rows fetched by queries |
| `tup_inserted` / `tup_updated` / `tup_deleted` | DML counts |
| `stats_reset` | When stats were last reset |

**Cache hit ratio:** `blks_hit / (blks_hit + blks_read)` — should be above 99%.

---

## Table Statistics Views

### pg_stat_user_tables
One row per user table. Core view for vacuum and bloat monitoring.

| Column | Description |
|--------|-------------|
| `schemaname`, `relname` | Table identity |
| `seq_scan` | Full table scans since stats reset |
| `seq_tup_read` | Rows read by full scans |
| `idx_scan` | Index scans |
| `idx_tup_fetch` | Rows fetched by index scans |
| `n_tup_ins`, `n_tup_upd`, `n_tup_del` | DML counts |
| `n_live_tup` | Estimated live rows |
| `n_dead_tup` | Estimated dead rows (MVCC garbage) |
| `n_mod_since_analyze` | Changes since last ANALYZE |
| `n_ins_since_vacuum` | Inserts since last VACUUM |
| `last_vacuum`, `last_autovacuum` | Last vacuum timestamps |
| `last_analyze`, `last_autoanalyze` | Last analyze timestamps |
| `vacuum_count`, `autovacuum_count` | How many times vacuumed |

### pg_statio_user_tables
Buffer cache statistics per table.

| Column | Description |
|--------|-------------|
| `heap_blks_read` | Table blocks read from disk |
| `heap_blks_hit` | Table blocks served from cache |
| `idx_blks_read` | Index blocks read from disk |
| `idx_blks_hit` | Index blocks served from cache |
| `toast_blks_read`, `toast_blks_hit` | TOAST I/O |

---

## Index Statistics Views

### pg_stat_user_indexes
One row per user index.

| Column | Description |
|--------|-------------|
| `schemaname`, `tablename`, `indexname` | Index identity |
| `idx_scan` | Number of times index was used |
| `idx_tup_read` | Index entries scanned |
| `idx_tup_fetch` | Table rows fetched via index |

**Unused indexes:** `idx_scan = 0` since last stats reset. Note: stats reset after server restart.

### pg_statio_user_indexes
I/O statistics per index.

| Column | Description |
|--------|-------------|
| `idx_blks_read` | Index blocks read from disk |
| `idx_blks_hit` | Index blocks served from cache |

---

## Background Process Views

### pg_stat_bgwriter
Cumulative checkpoint and background writer statistics.

| Column | Description |
|--------|-------------|
| `checkpoints_timed` | Scheduled checkpoints (at `checkpoint_timeout`) |
| `checkpoints_req` | Requested checkpoints (because `max_wal_size` reached) |
| `checkpoint_write_time` | Time writing dirty buffers to disk (ms) |
| `checkpoint_sync_time` | Time syncing files to disk (ms) |
| `buffers_checkpoint` | Buffers written by checkpoints |
| `buffers_clean` | Buffers written by bgwriter |
| `buffers_backend` | Buffers written directly by backends (bad sign) |
| `buffers_backend_fsync` | Times backend called fsync (critical alert) |
| `buffers_alloc` | Buffers allocated |

**Alert conditions:**
- `buffers_backend_fsync > 0` — checkpoint cannot keep up; backend forced to write WAL
- `checkpoints_req / (checkpoints_timed + checkpoints_req) > 10%` — `max_wal_size` is too small

### pg_stat_wal (PostgreSQL 14+)
WAL generation statistics.

| Column | Description |
|--------|-------------|
| `wal_records` | WAL records generated |
| `wal_fpi` | Full page images written (full_page_writes) |
| `wal_bytes` | Total bytes of WAL generated |
| `wal_buffers_full` | Times WAL buffers were full |
| `wal_write` | WAL write operations |
| `wal_sync` | WAL sync operations |

---

## Replication Views

### pg_stat_replication (primary only)
One row per connected standby.

| Column | Description |
|--------|-------------|
| `client_addr` | Standby IP address |
| `application_name` | Standby name (from `primary_conninfo`) |
| `state` | `streaming`, `catchup`, `backup`, `startup` |
| `sent_lsn` | Last WAL position sent to standby |
| `write_lsn` | Last WAL position written to standby disk |
| `flush_lsn` | Last WAL position flushed to standby disk |
| `replay_lsn` | Last WAL position replayed on standby |
| `write_lag`, `flush_lag`, `replay_lag` | Time lag at each stage |
| `sync_state` | `async`, `potential`, `sync`, `quorum` |

### pg_replication_slots
Replication slot state.

| Column | Description |
|--------|-------------|
| `slot_name` | Slot identifier |
| `slot_type` | `physical` or `logical` |
| `active` | Whether a process is consuming the slot |
| `restart_lsn` | Oldest WAL that must be retained |
| `wal_status` | `reserved`, `extended`, `unreserved`, `lost` (PG13+) |
| `safe_wal_size` | Bytes remaining before slot is lost (PG13+) |

**Alert:** A slot with `active = false` that retains a large amount of WAL will fill the disk.

---

## Progress Views

### pg_stat_progress_vacuum
One row per actively running VACUUM.

| Column | Description |
|--------|-------------|
| `relid` | Table being vacuumed |
| `phase` | Current phase: `scanning heap`, `vacuuming indexes`, `vacuuming heap`, `cleaning up indexes`, `truncating heap`, `performing final cleanup` |
| `heap_blks_total`, `heap_blks_scanned`, `heap_blks_vacuumed` | Block progress counters |
| `index_vacuum_count` | Number of index vacuum cycles completed |
| `num_dead_tuples` | Dead tuples found so far |
| `max_dead_tuples` | Maximum dead tuples before index vacuum triggered |

### pg_stat_progress_analyze
One row per running ANALYZE.

| Column | Description |
|--------|-------------|
| `relid` | Table being analyzed |
| `phase` | `acquiring sample rows` or `acquiring inherited sample rows` |
| `sample_blks_total`, `sample_blks_scanned` | Block progress |

---

## Lock Views

### pg_locks
One row per lock. Join with `pg_stat_activity` for query context.

| Column | Description |
|--------|-------------|
| `locktype` | `relation`, `tuple`, `transactionid`, `advisory`, etc. |
| `relation` | OID of locked relation (if applicable) |
| `mode` | `AccessShareLock`, `RowShareLock`, `RowExclusiveLock`, `ShareUpdateExclusiveLock`, `ShareLock`, `ShareRowExclusiveLock`, `ExclusiveLock`, `AccessExclusiveLock` |
| `granted` | `false` if waiting for lock |
| `pid` | Backend holding or waiting for the lock |

**Lock mode compatibility:** `AccessExclusiveLock` (ALTER TABLE, VACUUM FULL, LOCK TABLE) blocks all other locks.

### pg_blocking_pids(pid)
Function that returns an array of PIDs blocking a given PID. Use in `WHERE cardinality(pg_blocking_pids(pid)) > 0` to find blocked sessions.

---

## Useful Functions

| Function | Returns |
|----------|---------|
| `pg_postmaster_start_time()` | Timestamp when server started |
| `pg_database_size(name)` | Database size in bytes |
| `pg_relation_size(oid)` | Table/index size (main fork) |
| `pg_total_relation_size(oid)` | Table + indexes + TOAST |
| `pg_indexes_size(oid)` | All indexes for a table |
| `pg_size_pretty(bigint)` | Human-readable size string |
| `pg_current_wal_lsn()` | Current WAL write position |
| `pg_walfile_name(lsn)` | WAL file name for an LSN |
| `pg_wal_lsn_diff(lsn1, lsn2)` | Difference in bytes between two LSNs |
| `pg_blocking_pids(pid)` | PIDs blocking a given backend |
| `pg_terminate_backend(pid)` | Terminate a backend process |
| `pg_cancel_backend(pid)` | Cancel a backend's current query |
| `pg_reload_conf()` | Reload configuration files |
| `pg_stat_reset()` | Reset per-database statistics |
| `age(xid)` | Transaction age (distance from current XID) |
