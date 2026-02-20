# PostgreSQL Replication Examples

Practical scenarios and how to implement PostgreSQL replication best practices.

## Scenario 1: Monitoring Replication Lag

### Diagnosis
1.  **Monitor Replication Status:** Use `pg_stat_replication` to identify active standby servers and their current status.
2.  **Observation:** A standby `standby_1` has high replication lag in bytes.
```sql
SELECT
    application_name,
    pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) / 1024 / 1024 AS lag_mb
FROM pg_stat_replication;
```
3.  **Action:** Check the network bandwidth and disk latency on the standby.

### Procedure
1.  **Check Network Bandwidth:** Use tools like `iperf` to measure the network throughput between the primary and standby.
2.  **Check Disk Latency on Standby:** Use tools like `iostat` or `iotop` to identify slow disks or high I/O workloads.
3.  **Adjust Replication Settings:** Increase `max_wal_senders` and `wal_keep_size` in `postgresql.conf` if needed to avoid WAL segment deletion.
**Result:** Replication lag is reduced, ensuring that the standby is up-to-date and ready for failover.

## Scenario 2: Setting Up a Logical Replication Slot

### Diagnosis
1.  **Goal:** Replicate a specific database `mydb` to a secondary server using logical replication.

### Procedure
1.  **Create a Logical Replication Slot (Primary):**
```sql
SELECT pg_create_logical_replication_slot('mydb_slot', 'test_decoding');
```
2.  **Configure the Subscription (Secondary):**
```sql
CREATE SUBSCRIPTION mydb_sub
CONNECTION 'host=primary_host dbname=mydb user=repl_user password=repl_password'
PUBLICATION mydb_pub;
```
3.  **Verify the Subscription Status (Secondary):**
```sql
SELECT * FROM pg_stat_subscription;
```
**Result:** The specific database `mydb` is now being replicated to the secondary server using logical replication.

## Scenario 3: Performing a Manual Failover

### Diagnosis
1.  **Goal:** Promote the most up-to-date standby to become the new primary after a primary failure.

### Procedure
1.  **Stop the Primary Server:**
```bash
sudo systemctl stop postgresql
```
2.  **Promote the Standby Server:**
```bash
# PostgreSQL 12+
sudo -u postgres pg_ctl promote -D /var/lib/postgresql/data
```
3.  **Verify the Promotion:** Check the PostgreSQL logs for "server promoted" and "server is ready to accept connections" messages.
4.  **Update the Application and Load Balancer:** Point them to the new primary.
**Result:** The standby is successfully promoted to the new primary, minimizing downtime and data loss.

## Scenario 4: Implementing Patroni for Automated HA

### Diagnosis
1.  **Goal:** Implement Patroni for reliable, production-ready HA with automated failover and re-configuration.

### Procedure
1.  **Set Up a Distributed Configuration Store (DCS):** (e.g., etcd, Consul, ZooKeeper).
2.  **Install Patroni on All PostgreSQL Nodes:**
3.  **Configure Patroni (All Nodes):** Set the DCS configuration, PostgreSQL settings, and Patroni parameters.
4.  **Start Patroni (All Nodes):**
```bash
sudo systemctl start patroni
```
5.  **Monitor the Patroni Cluster Status:**
```bash
patronictl -c /etc/patroni/patroni.yml list
```
**Result:** Patroni will automatically manage the cluster of PostgreSQL servers, providing reliable, production-ready HA with automated failover and re-configuration.
