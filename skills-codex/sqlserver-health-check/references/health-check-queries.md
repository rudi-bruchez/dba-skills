# SQL Server Health Check Queries

## Instance Baseline
```sql
SELECT @@VERSION AS sql_version;
SELECT sqlserver_start_time FROM sys.dm_os_sys_info;
```

## Top Waits
```sql
SELECT TOP (20) wait_type, wait_time_ms, signal_wait_time_ms
FROM sys.dm_os_wait_stats
WHERE wait_type NOT LIKE 'SLEEP%'
ORDER BY wait_time_ms DESC;
```

## Blocking Snapshot
```sql
SELECT session_id, blocking_session_id, wait_type, wait_time, status
FROM sys.dm_exec_requests
WHERE blocking_session_id <> 0;
```

## Backup Freshness
```sql
SELECT d.name,
       MAX(CASE WHEN b.type='D' THEN b.backup_finish_date END) AS last_full,
       MAX(CASE WHEN b.type='I' THEN b.backup_finish_date END) AS last_diff,
       MAX(CASE WHEN b.type='L' THEN b.backup_finish_date END) AS last_log
FROM sys.databases d
LEFT JOIN msdb.dbo.backupset b ON b.database_name = d.name
GROUP BY d.name;
```
