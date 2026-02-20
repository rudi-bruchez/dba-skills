# PG Stat Reference

A guide to the most important PostgreSQL system views and extensions for monitoring.

## 1. System Views

| View Name | Description | Key Columns |
|---|---|---|
| `pg_stat_activity` | Information on all backend processes and current queries. | `pid`, `state`, `wait_event`, `query`, `query_start` |
| `pg_stat_database` | Instance-wide performance statistics (commits, rollbacks, cache hits). | `datname`, `xact_commit`, `xact_rollback`, `blks_hit`, `blks_read` |
| `pg_stat_user_tables` | Table-specific statistics (scans, inserts, updates, dead tuples). | `relname`, `n_live_tup`, `n_dead_tup`, `last_vacuum`, `last_autovacuum` |
| `pg_stat_user_indexes` | Index usage statistics. | `relname`, `indexrelname`, `idx_scan`, `idx_tup_read`, `idx_tup_fetch` |
| `pg_statio_user_tables` | I/O-specific statistics for tables (blocks hit vs read). | `relname`, `heap_blks_read`, `heap_blks_hit`, `idx_blks_read`, `idx_blks_hit` |
| `pg_stat_wal_receiver` | Status of the WAL receiver (if replication is active). | `status`, `receive_start_lsn`, `receive_start_tli` |
| `pg_stat_replication` | Status of the replication (on the primary). | `usename`, `application_name`, `client_addr`, `state`, `write_lag` |

## 2. Essential Extensions

| Extension Name | Description | Key Features |
|---|---|---|
| `pg_stat_statements` | Aggregates performance data for all executed SQL statements. | `query`, `calls`, `total_exec_time`, `mean_exec_time`, `rows` |
| `pgstattuple` | Provides detailed information about bloat in tables and indexes. | `pgstattuple('table_name')`, `pgstatindex('index_name')` |
| `pgaudit` | Provides session and/or object-level logging. | `pgaudit.log = 'ddl, write'`, `pgaudit.log_catalog = on` |
| `pg_buffercache` | Provides visibility into what is in the shared buffer cache. | `pg_buffercache_summary()`, `pg_buffercache_pages()` |
| `pg_prewarm` | Allows pre-loading data into the buffer cache for faster performance. | `pg_prewarm('table_name')` |

## Best Practices
- **Enable `pg_stat_statements`:** Add it to `shared_preload_libraries` in `postgresql.conf`.
- **Reset Statistics Regularly:** Use `pg_stat_reset()` or `pg_stat_reset_single_table_counters(oid)` as needed for baselines.
- **Monitoring Tools:** Use Prometheus/Grafana or PMM for real-time dashboards based on these views.
