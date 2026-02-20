# PostgreSQL Health Check & Diagnostics Research

## Overview
PostgreSQL health checks focus on monitoring activity, identifying bottlenecks, and maintaining database hygiene (e.g., vacuuming). It leverages internal system views and external tools for comprehensive visibility.

## Key Performance Indicators (KPIs)
- **Active Connections:** Track `state` and `query` in `pg_stat_activity`.
- **Transaction Rate:** Monitor `xact_commit` and `xact_rollback` in `pg_stat_database`.
- **Cache Hit Ratio:** Percentage of data blocks found in `shared_buffers` vs. read from disk. Aim for >95% for OLTP.
- **Wait Events:** Monitor what queries are waiting for (e.g., `LWLock`, `IO`, `Lock`, `BufferPin`) via `pg_stat_activity`.
- **Table & Index Bloat:** Dead tuples that take up space but don't hold data. Can significantly slow down scans and increase storage.

## Essential System Views & Extensions
- `pg_stat_activity`: Real-time view of all backend processes and current queries.
- `pg_stat_database`: Instance-wide statistics (commits, rollbacks, cache hits).
- `pg_stat_user_tables`: Table-specific stats (scans, inserts, updates, dead tuples).
- `pg_stat_user_indexes`: Index usage statistics.
- `pg_stat_statements`: (Extension) Aggregates performance data for all executed SQL statements.
- `pgstattuple`: (Extension) Provides detailed information about bloat in tables and indexes.

## Maintenance & Housekeeping
- **Vacuuming:** Reclaims space from deleted or updated rows (dead tuples).
- **Autovacuum:** Background process that handles vacuuming and analyzing automatically.
- **Tuning Autovacuum:** Adjusting `autovacuum_vacuum_scale_factor` and `autovacuum_analyze_scale_factor` to be more aggressive for busy tables.
- **REINDEX:** Rebuilds indexes to remove bloat. `REINDEX CONCURRENTLY` (PG 12+) is recommended to avoid locking tables.

## Logging & Diagnostics
- **Slow Query Logging:** Use `log_min_duration_statement` to log queries exceeding a threshold (e.g., 500ms).
- **pgBadger:** A powerful log analyzer that produces HTML reports with detailed performance insights.
- **Explain & Analyze:** Use `EXPLAIN (ANALYZE, BUFFERS)` to diagnose a single slow query.

## Best Practices (2024-2025)
- **Monitoring Tools:** Use Prometheus/Grafana or PMM for real-time dashboards.
- **Connection Pooling:** Use `pgBouncer` to manage a large number of client connections efficiently.
- **Bloat Monitoring:** Regularly check for bloat using `pg_bloat_check` or similar scripts.
- **Transaction Duration Alerts:** Set up alerts for long-running transactions that prevent autovacuum from doing its job.

## Key SQL Queries
- `SELECT * FROM pg_stat_activity WHERE state != 'idle';`
- `SELECT relname, n_live_tup, n_dead_tup, last_vacuum, last_autovacuum FROM pg_stat_user_tables;`
- `SELECT * FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 10;`

## References
- [PostgreSQL Documentation: Monitoring Performance](https://www.postgresql.org/docs/current/monitoring.html)
- [pganalyze: PostgreSQL Guide](https://pganalyze.com/docs)
- [Postgres Checkup: Health Check Tool](https://github.com/postgres-ai/postgres-checkup)
