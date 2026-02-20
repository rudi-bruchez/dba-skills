# SQL Server Backup & Recovery Examples

Practical scenarios and how to perform common backup and recovery tasks.

## Scenario 1: Accidental Data Deletion

### Diagnosis
1.  **Issue:** A user accidentally deleted several rows from the `Orders` table at 2:15 PM today.
2.  **Goal:** Restore the database to 2:10 PM today, just before the deletion.

### Recovery Procedure
Restore the database using Point-in-Time Recovery (PITR).
```sql
-- 1. Restore the most recent Full Backup (e.g., from last night)
RESTORE DATABASE [MyDatabase_Restore]
FROM DISK = 'C:\Backups\MyDatabase_Full.bak'
WITH MOVE 'MyDatabase_Data' TO 'C:\Data\MyDatabase_Restore.mdf',
     MOVE 'MyDatabase_Log' TO 'D:\Logs\MyDatabase_Restore.ldf',
     NORECOVERY, REPLACE;

-- 2. Restore the most recent Transaction Log(s) up to the target time
RESTORE LOG [MyDatabase_Restore]
FROM DISK = 'C:\Backups\MyDatabase_Log.trn'
WITH STOPAT = '2023-10-27 14:10:00', RECOVERY;
```
**Result:** The database `MyDatabase_Restore` is now at its 2:10 PM state. The deleted data can be verified and copied back to the original database.

## Scenario 2: Corruption in a Specific Data Page

### Diagnosis
1.  **Issue:** A query fails with an error indicating a corrupted data page (e.g., Error 823 or 824).
2.  **Action:** Identify the damaged page from the error message or `msdb.dbo.suspect_pages`.

### Recovery Procedure
Perform a Page-Level Restore (Enterprise Edition only).
```sql
-- 1. Identify the damaged page
SELECT database_id, file_id, page_id, event_type, last_update_date
FROM msdb.dbo.suspect_pages;

-- 2. Restore the specific page from the full backup
RESTORE DATABASE [MyDatabase]
PAGE = '1:1234' -- file:page
FROM DISK = 'C:\Backups\MyDatabase_Full.bak'
WITH NORECOVERY;

-- 3. Restore all transaction log backups taken since the full backup
RESTORE LOG [MyDatabase]
FROM DISK = 'C:\Backups\MyDatabase_Log.trn'
WITH RECOVERY;
```
**Result:** Only the corrupted page is restored from the backup, and subsequent transaction logs are replayed to bring it back to its current state. The rest of the database remains online.

## Scenario 3: Moving a Database to a New Server

### Diagnosis
1.  **Goal:** Move the `MyDatabase` database to a new server using backup and restore.

### Procedure
1.  **Backup on Source Server:**
```sql
BACKUP DATABASE [MyDatabase]
TO DISK = '\NetworkShare\Backups\MyDatabase_Move.bak'
WITH CHECKSUM, COMPRESSION, STATS = 10;
```
2.  **Restore on Target Server:**
```sql
RESTORE DATABASE [MyDatabase]
FROM DISK = '\NetworkShare\Backups\MyDatabase_Move.bak'
WITH MOVE 'MyDatabase_Data' TO 'C:\Data\MyDatabase.mdf',
     MOVE 'MyDatabase_Log' TO 'D:\Logs\MyDatabase.ldf',
     RECOVERY, STATS = 10;
```
**Result:** The database is now restored and ready for use on the new server.

## Scenario 4: Performing a Copy-Only Backup

### Diagnosis
1.  **Goal:** Take a one-off backup of a database for a specific task without disrupting the regular backup schedule or LSN chain.

### Procedure
```sql
BACKUP DATABASE [MyDatabase]
TO DISK = 'C:\Backups\MyDatabase_OneOff.bak'
WITH COPY_ONLY, CHECKSUM, COMPRESSION, STATS = 10;
```
**Result:** A full backup is created, but the LSN chain is not affected, meaning that regular transaction log and differential backups can still be taken as scheduled.
