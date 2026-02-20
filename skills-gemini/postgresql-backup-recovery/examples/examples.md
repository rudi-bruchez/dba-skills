# PostgreSQL Backup & Recovery Examples

Practical scenarios and how to perform common backup and recovery tasks.

## Scenario 1: Accidental Data Deletion

### Diagnosis
1.  **Issue:** A user accidentally deleted several rows from the `orders` table at 2:15 PM today.
2.  **Goal:** Restore the database to 2:10 PM today, just before the deletion.

### Recovery Procedure
Restore the database using Point-in-Time Recovery (PITR).
```bash
# 1. Stop the PostgreSQL server
sudo systemctl stop postgresql

# 2. Extract the base backup into the data directory
sudo -u postgres tar -xzf /path/to/backup/base_backup.tar.gz -C /var/lib/postgresql/data

# 3. Create a recovery.signal file (PG 12+)
sudo -u postgres touch /var/lib/postgresql/data/recovery.signal

# 4. Configure restore_command and recovery_target_time in postgresql.conf
# restore_command = 'cp /path/to/archive/%f %p'
# recovery_target_time = '2023-10-27 14:10:00'

# 5. Start the PostgreSQL server
sudo systemctl start postgresql
```
**Result:** The database is now at its 2:10 PM state. The deleted data can be verified and copied back to the original database.

## Scenario 2: Corruption in a Specific Table

### Diagnosis
1.  **Issue:** A query fails with an error indicating an "invalid page in block" for the `users` table.
2.  **Action:** Identify the damage from the PostgreSQL log files.

### Recovery Procedure
Perform a full cluster restore from a base backup and replay WAL archives.
```bash
# 1. Stop the PostgreSQL server
sudo systemctl stop postgresql

# 2. Restore the entire database cluster from a base backup
sudo -u postgres tar -xzf /path/to/backup/base_backup.tar.gz -C /var/lib/postgresql/data

# 3. Replay WAL archives to bring the cluster to its current state
# recovery.signal should be created if recovery is needed.
# Start the server and check logs for "consistent recovery state reached."
sudo systemctl start postgresql
```
**Result:** Only the corrupted page is restored from the backup, and subsequent WAL segments are replayed to bring it back to its current state. The rest of the database remains online.

## Scenario 3: Moving a Database to a New Server

### Diagnosis
1.  **Goal:** Move the `mydb` database to a new server using a logical dump.

### Procedure
1.  **Backup on Source Server:**
```bash
pg_dump -U postgres -d mydb -F c -f mydb_move.dump
```
2.  **Restore on Target Server:**
```bash
pg_restore -U postgres -d mydb mydb_move.dump
```
**Result:** The database is now restored and ready for use on the new server.

## Scenario 4: Performing a Copy-Only Backup

### Diagnosis
1.  **Goal:** Take a one-off backup of a database for a specific task without disrupting the regular backup schedule or WAL chain.

### Procedure
```bash
pg_dump -U postgres -d mydb -F c -f mydb_oneoff.dump
```
**Result:** A logical backup is created, but the WAL chain is not affected, meaning that regular WAL archiving and physical backups can still be taken as scheduled.
