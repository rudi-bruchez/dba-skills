# Recovery Scenarios

Common PostgreSQL restore scenarios and their recovery procedures.

## 1. Restoring a Single Database (pg_restore)
Restore a database from a custom format dump created with `pg_dump -F c`.
```bash
pg_restore -U [user] -d [db_name] -C db_backup.dump
```
- **-C:** Creates the database before restoring data.
- **-d [db_name]:** The target database name.

## 2. Restoring to a Different Schema
```bash
pg_restore -U [user] -d [db_name] --schema=[new_schema] db_backup.dump
```

## 3. Restoring to a Different Database Version
1.  **Backup using the newer version's `pg_dump`:**
    ```bash
    pg_dump -U [user] -h [old_host] -d [db_name] -F c -f db_backup.dump
    ```
2.  **Restore using the newer version's `pg_restore`:**
    ```bash
    pg_restore -U [user] -d [db_name] db_backup.dump
    ```

## 4. Recovering from a Corrupted Data Page
1.  **Identify the damage:** Check the PostgreSQL logs for "invalid page in block" errors.
2.  **Restore the entire database cluster:** Perform a full cluster restore from a base backup and replay WAL archives.
3.  **Use `pg_checksums` (PG 11+):** Enable checksums on the cluster to detect corruption early.

## 5. Recovering from a Deleted Table
1.  **Goal:** Recover the `Orders` table that was deleted at 3:00 PM.
2.  **Procedure:** Perform PITR restore to 2:59 PM.
    - Extract base backup.
    - Create `recovery.signal`.
    - Set `recovery_target_time = '2023-10-27 14:59:00'`.
    - Start the server.
    - Export the `Orders` table and import it back into the original database.

## Best Practices
- **Verify Backups Regularly:** Perform "restore tests" to ensure backups and WAL segments are valid.
- **Offsite Storage:** Store backups and WAL segments in geographically separate locations.
- **Use pgBackRest:** It simplifies recovery significantly with built-in commands like `pgbackrest restore`.
- **Automated Restore Testing:** Automate the process of restoring backups to a non-production environment regularly.

## References
- [PostgreSQL Documentation: Backup and Restore](https://www.postgresql.org/docs/current/backup.html)
- [pgBackRest: Documentation](https://pgbackrest.org/user-guide.html)
- [Barman: Documentation](https://www.pgbarman.org/documentation/)
