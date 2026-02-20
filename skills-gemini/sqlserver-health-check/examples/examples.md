# SQL Server Health Check Examples

Practical scenarios and how to interpret diagnostic results.

## Scenario 1: Sudden Performance Degradation

### Diagnosis
1.  **Check Active Sessions:** Run `sp_WhoIsActive` or `sys.dm_exec_requests`.
2.  **Wait Statistics:** Found high `LCK_M_X` (exclusive lock) waits.
3.  **Lead Blocker:** Identify the session holding the lock.
```sql
SELECT session_id, blocking_session_id, wait_type, wait_time, text
FROM sys.dm_exec_requests
CROSS APPLY sys.dm_exec_sql_text(sql_handle)
WHERE blocking_session_id <> 0;
```
4.  **Action:** A large `DELETE` operation was holding an exclusive lock on the `Sales` table, blocking several `SELECT` queries.
5.  **Solution:** Batch the delete operation or schedule it during off-peak hours. Use `NOLOCK` for non-critical reads if appropriate.

## Scenario 2: High CPU Utilization (> 90%)

### Diagnosis
1.  **Top CPU Queries:** Identify the queries causing high pressure.
```sql
SELECT TOP 5
    st.text,
    qs.total_worker_time / 1000 AS TotalCPUTime_ms,
    qs.execution_count,
    qp.query_plan
FROM sys.dm_exec_query_stats AS qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) AS st
CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) AS qp
ORDER BY qs.total_worker_time DESC;
```
2.  **Observation:** A query was performing a full `Index Scan` instead of an `Index Seek`.
3.  **Root Cause:** Missing index on the column used in the `WHERE` clause.
4.  **Solution:** Create the recommended index found in the `sys.dm_db_missing_index_details` DMV.

## Scenario 3: Low Page Life Expectancy (PLE)

### Diagnosis
1.  **Memory Health:** PLE was consistently < 100 seconds.
2.  **Buffer Cache Hit Ratio:** Dropped to 88%.
3.  **Action:** Check which database is using the most buffer pool memory.
```sql
SELECT
    DB_NAME(database_id) AS DatabaseName,
    COUNT(*) * 8 / 1024 AS MBUsed
FROM sys.dm_os_buffer_descriptors
GROUP BY database_id
ORDER BY MBUsed DESC;
```
4.  **Solution:** Identified a database with a very large table being scanned frequently. Optimized queries and added RAM to the instance.

## Scenario 4: High I/O Latency

### Diagnosis
1.  **File Stats:** Read latency on the data file was > 100ms.
2.  **Wait Stats:** High `PAGEIOLATCH_SH` waits.
3.  **Investigation:** Found that the storage array was performing a backup/maintenance task during business hours.
4.  **Solution:** Reschedule maintenance tasks to off-peak periods or upgrade to faster storage (SSD/NVMe).
