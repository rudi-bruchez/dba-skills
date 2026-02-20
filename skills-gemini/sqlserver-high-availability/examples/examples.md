# SQL Server High Availability Examples

Practical scenarios and how to implement Always On Availability Group (AG) best practices.

## Scenario 1: Monitoring Replication Lag

### Diagnosis
1.  **Monitor Replication Status:** Use `sys.dm_hadr_database_replica_states` to identify active standby replicas and their current status.
2.  **Observation:** A standby replica `SQL2` has a large `log_send_queue_size`.
```sql
SELECT
    ar.replica_server_name AS ReplicaName,
    drrs.log_send_queue_size / 1024 AS log_send_queue_mb
FROM sys.availability_groups AS ag
JOIN sys.availability_replicas AS ar ON ag.group_id = ar.group_id
JOIN sys.dm_hadr_database_replica_states AS drrs ON ar.replica_id = drrs.replica_id;
```
3.  **Action:** Check the network bandwidth and disk latency on the secondary.

### Procedure
1.  **Check Network Bandwidth:** Use tools like `iperf` to measure the network throughput between the primary and secondary.
2.  **Check Disk Latency on Secondary:** Use tools like `iostat` or `iotop` to identify slow disks or high I/O workloads.
3.  **Adjust Replication Settings:** Increase `max_wal_senders` and `wal_keep_size` in `postgresql.conf` if needed to avoid WAL segment deletion. (Wait, this is PostgreSQL! I mean SQL Server settings!)
Actually, for SQL Server:
- Check for `HADR_SYNC_COMMIT` waits on the primary.
- Optimize log file I/O on the secondary.
- Ensure the network is not saturated.

**Result:** Replication lag is reduced, ensuring that the standby is up-to-date and ready for failover.

## Scenario 2: Setting Up a Read-Only Listener

### Diagnosis
1.  **Goal:** Offload read-intent queries from the primary replica to a secondary replica.

### Procedure
1.  **Configure Read-Only Routing URL (All Replicas):**
```sql
ALTER AVAILABILITY GROUP [AGName]
MODIFY REPLICA ON 'SQL1' WITH (SECONDARY_ROLE (READ_ONLY_ROUTING_URL = 'TCP://SQL1.domain.com:1433'));
ALTER AVAILABILITY GROUP [AGName]
MODIFY REPLICA ON 'SQL2' WITH (SECONDARY_ROLE (READ_ONLY_ROUTING_URL = 'TCP://SQL2.domain.com:1433'));
```
2.  **Configure Read-Only Routing List (Primary):**
```sql
ALTER AVAILABILITY GROUP [AGName]
MODIFY REPLICA ON 'SQL1' WITH (PRIMARY_ROLE (READ_ONLY_ROUTING_LIST = ('SQL2', 'SQL1')));
```
3.  **Update Connection String:** Set `ApplicationIntent=ReadOnly` and `MultiSubnetFailover=True`.
**Result:** Read-intent queries are now automatically directed to the secondary replica `SQL2`, offloading the primary.

## Scenario 3: Performing a Manual Failover

### Diagnosis
1.  **Goal:** Perform a planned failover to verify the cluster's readiness and ensure no data loss.

### Procedure
1.  **Stop the primary instance or disconnect it from the network.**
2.  **Verify that the standby server has been promoted to the new primary.**
```sql
ALTER AVAILABILITY GROUP [AGName] FAILOVER;
```
3.  **Check for data loss if using asynchronous replication.**
4.  **Re-configure the old primary as a standby after fixing the issue.**
**Result:** The standby is successfully promoted to the new primary, minimizing downtime and data loss.

## Scenario 4: Resolving a "Split-Brain" Scenario

### Diagnosis
1.  **Goal:** Force the cluster back to its correct state after a "split-brain" scenario.

### Procedure
1.  **Identify the true primary node.**
2.  **Stop the other node.**
3.  **Force the failover on the correct node:**
```sql
ALTER AVAILABILITY GROUP [AGName] FORCE_FAILOVER_ALLOW_DATA_LOSS;
```
**Result:** The cluster is returned to a consistent state, though some data loss may have occurred.
