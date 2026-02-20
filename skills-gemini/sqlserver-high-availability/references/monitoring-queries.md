# Monitoring Queries

A collection of essential T-SQL queries for diagnosing Always On Availability Group (AG) status and health.

## 1. Check Availability Group Status
Identify active standby servers and their current status.
```sql
SELECT
    ag.name AS AGName,
    ar.replica_server_name AS ReplicaName,
    ar.availability_mode_desc,
    rs.role_desc,
    rs.operational_state_desc,
    rs.connected_state_desc,
    rs.recovery_health_desc,
    rs.synchronization_health_desc
FROM sys.availability_groups AS ag
JOIN sys.availability_replicas AS ar ON ag.group_id = ar.group_id
JOIN sys.dm_hadr_availability_replica_states AS rs ON ar.replica_id = rs.replica_id;
```

## 2. Check Database Replication Status
Verify if the standby is currently replaying and re-synchronizing WAL data.
```sql
SELECT
    ag.name AS AGName,
    ar.replica_server_name AS ReplicaName,
    drcs.database_name,
    drcs.synchronization_state_desc,
    drcs.is_failover_ready
FROM sys.availability_groups AS ag
JOIN sys.availability_replicas AS ar ON ag.group_id = ar.group_id
JOIN sys.dm_hadr_database_replica_cluster_states AS drcs ON ar.replica_id = drcs.replica_id;
```

## 3. Measure Replication Lag (Primary)
Measure the distance between the primary and standby in bytes or time.
```sql
SELECT
    ar.replica_server_name AS ReplicaName,
    drrs.last_sent_time,
    drrs.last_received_time,
    drrs.last_hardened_time,
    drrs.last_redone_time,
    drrs.log_send_queue_size / 1024 AS log_send_queue_mb,
    drrs.redo_queue_size / 1024 AS redo_queue_mb
FROM sys.availability_groups AS ag
JOIN sys.availability_replicas AS ar ON ag.group_id = ar.group_id
JOIN sys.dm_hadr_database_replica_states AS drrs ON ar.replica_id = drrs.replica_id;
```

## 4. Check for Synchronization Bottlenecks
Identify common wait events (e.g., `HADR_SYNC_COMMIT`) that indicate network or disk latency issues.
```sql
SELECT
    wait_type,
    wait_time_ms / 1000.0 AS WaitS,
    percentage = CAST(100.0 * wait_time_ms / SUM(wait_time_ms) OVER() AS DECIMAL(5,2))
FROM sys.dm_os_wait_stats
WHERE wait_type IN ('HADR_SYNC_COMMIT', 'HADR_LOGCAPTURE_WAIT', 'HADR_NOTIFICATION_DEQUEUE')
ORDER BY wait_time_ms DESC;
```

## 5. View Failover History
Check for failover events and errors in the SQL Server Error Log and Extended Events.

## References
- [Microsoft Documentation: Monitoring Availability Groups](https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/monitor-availability-groups-transact-sql)
- [Brent Ozar's Always On Availability Groups - Health Check](https://www.brentozar.com/sql/alwayson-availability-groups-health-check/)
- [SQLskills: HADR_SYNC_COMMIT Wait Type](https://www.sqlskills.com/help/waits/hadr_sync_commit/)
---
