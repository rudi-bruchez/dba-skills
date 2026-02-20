# Replication Diagnostics

A collection of essential SQL queries for diagnosing PostgreSQL replication status and health.

## 1. Check Replication Status (Primary)
Identify active standby servers and their current status.
```sql
SELECT
    application_name,
    client_addr,
    state,
    sync_state,
    (pg_current_wal_lsn() - replay_lsn) / 1024 / 1024 AS lag_mb
FROM pg_stat_replication;
```

## 2. Check Replication Status (Standby)
Verify if the standby is currently receiving and replaying WAL data.
```sql
SELECT
    status,
    receive_start_lsn,
    receive_start_tli,
    latest_end_lsn,
    latest_end_time
FROM pg_stat_wal_receiver;
```

## 3. Measure Replication Lag (Primary)
Measure the distance between the primary and standby in bytes or time.
```sql
SELECT
    application_name,
    pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag_bytes,
    pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) / 1024 / 1024 AS lag_mb
FROM pg_stat_replication;
```

## 4. Check for Replication Slots
Replication slots ensure that the primary doesn't delete WAL segments until they are consumed by the standby.
```sql
SELECT
    slot_name,
    slot_type,
    active,
    (pg_current_wal_lsn() - restart_lsn) / 1024 / 1024 AS retained_wal_mb
FROM pg_get_replication_slots();
```

## 5. View Replication History
Check for replication failures and re-connection attempts in the PostgreSQL log files.

## References
- [PostgreSQL Documentation: Monitoring Replication](https://www.postgresql.org/docs/current/monitoring-stats.html#MONITORING-STATS-REPLICATION-VIEWS)
- [pganalyze: Guide to PostgreSQL Replication Lag](https://pganalyze.com/blog/postgresql-replication-lag)
- [Prometheus PostgreSQL Exporter](https://github.com/prometheus-community/postgres_exporter)
