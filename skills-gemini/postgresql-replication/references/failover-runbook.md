# Failover Runbook

A step-by-step guide to performing a manual failover in a PostgreSQL streaming replication cluster.

## 1. Stop the Primary Server
This should be done cleanly to avoid data loss.
```bash
sudo systemctl stop postgresql
```

## 2. Promote the Standby Server
Promote the most up-to-date standby to become the new primary.
```bash
# PostgreSQL 12+
sudo -u postgres pg_ctl promote -D /var/lib/postgresql/data
```
Verify the promotion in the PostgreSQL logs for "server promoted" and "server is ready to accept connections" messages.

## 3. Re-configure the Old Primary as a Standby
After fixing the issue on the old primary:
1.  **Extract a base backup from the new primary.**
2.  **Configure the old primary as a standby.**
3.  **Start the old primary.**

## 4. Update the Application and Load Balancer
Ensure the application and load balancers are updated to point to the new primary.

## 5. Automated Failover (Patroni)
If using Patroni, the failover process is automated.
- **Failover:** Patroni will automatically detect a primary failure and promote a standby.
- **Re-configuration:** Patroni will automatically re-configure the old primary as a standby after it comes back online.

## References
- [PostgreSQL Documentation: High Availability and Replication](https://www.postgresql.org/docs/current/high-availability.html)
- [Patroni: Documentation](https://patroni.readthedocs.io/en/latest/)
- [repmgr: Documentation](https://repmgr.org/docs/current/)
- [pganalyze: Guide to PostgreSQL Failover](https://pganalyze.com/blog/postgresql-failover)
