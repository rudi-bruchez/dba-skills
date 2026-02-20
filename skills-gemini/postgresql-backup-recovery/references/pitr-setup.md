# PITR Setup Guide

Point-in-Time Recovery (PITR) allows you to restore a PostgreSQL database to a specific moment in time.

## 1. Requirements

- **Base Backup:** A physical backup created with `pg_basebackup` or `pgBackRest`.
- **WAL Archives:** Continuous transaction log (WAL) segments archived to a separate location.
- **`wal_level = replica`:** Enabled in `postgresql.conf`.
- **`archive_mode = on`:** Enabled in `postgresql.conf`.

## 2. Recovery Steps (PostgreSQL 12+)

To perform a PITR restore:

1.  **Stop the PostgreSQL Server:**
    ```bash
    sudo systemctl stop postgresql
    ```
2.  **Restore the Base Backup:**
    Extract the base backup into the PostgreSQL data directory.
    ```bash
    sudo -u postgres tar -xzf /path/to/backup/base_backup.tar.gz -C /var/lib/postgresql/data
    ```
3.  **Create a Recovery Signal File:**
    This tells PostgreSQL to start in recovery mode.
    ```bash
    sudo -u postgres touch /var/lib/postgresql/data/recovery.signal
    ```
4.  **Configure `postgresql.conf` (or `postgresql.auto.conf`):**
    Set the restore command and the target recovery time.
    ```conf
    restore_command = 'cp /path/to/archive/%f %p'
    recovery_target_time = '2023-10-27 14:30:00'
    ```
    - **%f:** The name of the WAL file.
    - **%p:** The path to where the file should be copied.
5.  **Start the PostgreSQL Server:**
    ```bash
    sudo systemctl start postgresql
    ```
6.  **Verify Recovery Status:**
    Check the log files to ensure recovery is complete and the database is at the correct state.

## 3. Best Practices

- **Test Regularly:** Perform PITR restore tests in a non-production environment.
- **Use pgBackRest:** It simplifies the PITR process significantly with built-in commands like `pgbackrest restore --target="2023-10-27 14:30:00"`.
- **Monitor WAL Archive Success:** Ensure that `archive_command` is succeeding and that WAL files are being archived regularly.
- **Verify Recovery Success:** Check the PostgreSQL logs for "restored log file" and "consistent recovery state reached" messages.

## References
- [PostgreSQL Documentation: Continuous Archiving and PITR](https://www.postgresql.org/docs/current/continuous-archiving.html)
- [pgBackRest: Documentation](https://pgbackrest.org/user-guide.html)
- [pganalyze: Guide to PostgreSQL PITR](https://pganalyze.com/blog/postgresql-pitr)
