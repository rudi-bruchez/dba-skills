# PostgreSQL Diagnostic SQL

## Active Sessions
```sql
SELECT pid, usename, state, wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE state <> 'idle';
```

## Database-Level Stats
```sql
SELECT datname, xact_commit, xact_rollback, blks_read, blks_hit
FROM pg_stat_database;
```

## Blocking Overview
```sql
SELECT pid, pg_blocking_pids(pid) AS blockers, query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
```
