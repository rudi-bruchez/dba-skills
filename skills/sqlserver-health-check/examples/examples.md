# SQL Server Health Check — Examples

---

## Scenario 1: Slow Server — Users Reporting Everything Is Slow

**Context:** Multiple users call in that queries that normally run in under 1 second are now taking 30+ seconds. No recent deployments.

**Step 1 — Check PLE immediately**
```sql
SELECT cntr_value AS ple
FROM sys.dm_os_performance_counters
WHERE counter_name = 'Page life expectancy'
  AND object_name LIKE '%Buffer Manager%';
-- Result: 142 (critical — threshold is > 300, modern guideline > 1000)
```

**Step 2 — Check top waits**
```sql
-- (run the full wait stats query from SKILL.md)
-- Top result: PAGEIOLATCH_SH — 87,000 ms total, 95% of waits
```

**Step 3 — Identify I/O hot files**
```sql
SELECT DB_NAME(fs.database_id) AS database_name, mf.physical_name,
       fs.io_stall_read_ms / NULLIF(fs.num_of_reads, 0) AS avg_read_ms
FROM sys.dm_io_virtual_file_stats(NULL, NULL) fs
JOIN sys.master_files mf ON fs.database_id = mf.database_id AND fs.file_id = mf.file_id
ORDER BY avg_read_ms DESC;
-- Result: SalesDB.mdf on D:\ — avg_read_ms = 178 ms (critical for SSD)
```

**Step 4 — Check buffer pool by database**
```sql
SELECT DB_NAME(database_id) AS db, COUNT(*) * 8 / 1024.0 AS buffer_mb
FROM sys.dm_os_buffer_descriptors WHERE database_id > 0
GROUP BY database_id ORDER BY buffer_mb DESC;
-- SalesDB: 12,480 MB (nearly entire buffer pool is this one database)
```

**Diagnosis:** Memory pressure — PLE crashed because a large scan in SalesDB evicted all cached pages. Find the query responsible.

**Step 5 — Find the offending query**
```sql
SELECT TOP 5 qs.total_logical_reads / qs.execution_count AS avg_reads,
       SUBSTRING(st.text, 1, 300) AS query_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
ORDER BY total_logical_reads DESC;
-- Result: SELECT * FROM dbo.OrderHistory WHERE YEAR(order_date) = 2023
--   avg_reads: 4,200,000 logical reads per execution
```

**Resolution:**
1. Fix the non-SARGable predicate: `WHERE order_date >= '2023-01-01' AND order_date < '2024-01-01'`.
2. Add a nonclustered index on `(order_date) INCLUDE (columns needed)`.
3. Verify PLE recovers after index creation and repeated execution.

---

## Scenario 2: Blocking — Specific Application Operation Hangs

**Context:** A sales order entry application hangs when multiple users try to save orders at the same time. The operation times out after 30 seconds.

**Step 1 — Confirm blocking exists**
```sql
SELECT COUNT(*) AS blocked_count
FROM sys.dm_exec_requests WHERE blocking_session_id > 0;
-- Result: 14 blocked sessions
```

**Step 2 — Find the head blocker**
```sql
SELECT r.blocking_session_id AS head_blocker,
       s.login_name, s.host_name, s.program_name,
       s.open_transaction_count,
       SUBSTRING(st.text, 1, 500) AS head_blocker_sql
FROM sys.dm_exec_sessions s
LEFT JOIN sys.dm_exec_requests r ON s.session_id = r.session_id
CROSS APPLY sys.dm_exec_sql_text(COALESCE(r.sql_handle, s.sql_handle)) st
WHERE s.session_id IN (
    SELECT DISTINCT blocking_session_id FROM sys.dm_exec_requests WHERE blocking_session_id > 0
)
  AND s.session_id NOT IN (
    SELECT session_id FROM sys.dm_exec_requests WHERE blocking_session_id > 0
);
-- Result: session_id 62, open_transaction_count = 1
-- SQL: UPDATE dbo.Inventory SET qty_available = qty_available - @qty WHERE product_id = @pid
-- program_name: SalesApp v2.1, host_name: APPSVR01
```

**Step 3 — Check what session 62 is waiting on**
```sql
SELECT wait_type, wait_time / 1000.0 AS wait_sec, last_wait_type
FROM sys.dm_exec_requests WHERE session_id = 62;
-- Result: wait_type = NULL (session is active but holding locks from previous statement)
-- This is a "sleeping" session holding an open transaction
```

**Immediate action:** The application opened a transaction, updated `Inventory`, and has not committed. Kill if production is impacted:
```sql
KILL 62;
```

**Root cause and permanent fix:**
- The application's `BEGIN TRANSACTION` wraps a long `SELECT` on `dbo.Products` before the `UPDATE`, holding an exclusive lock far longer than needed.
- Fix: Shorten the transaction — do the `SELECT` outside the transaction, then open a short transaction only for the `UPDATE`.
- Alternative: Enable RCSI to reduce read-write conflicts: `ALTER DATABASE [SalesDB] SET READ_COMMITTED_SNAPSHOT ON;`

---

## Scenario 3: Memory Pressure — High Number of Compiles and PLE Declining

**Context:** Monitoring alerts on declining PLE (now at 420, trending down). SQL Server compilations/sec counter is high.

**Step 1 — Check ad hoc plan cache pollution**
```sql
SELECT COUNT(*) AS single_use_plan_count,
       SUM(size_in_bytes) / 1024.0 / 1024.0 AS total_mb
FROM sys.dm_exec_cached_plans
WHERE usecounts = 1 AND objtype = 'Adhoc';
-- Result: 48,320 single-use plans, 2,341 MB consumed
```

**Step 2 — Sample some of the single-use plans**
```sql
SELECT TOP 10 SUBSTRING(st.text, 1, 200) AS sql_preview, size_in_bytes / 1024 AS size_kb
FROM sys.dm_exec_cached_plans cp
CROSS APPLY sys.dm_exec_sql_text(cp.plan_handle) st
WHERE usecounts = 1 AND objtype = 'Adhoc'
ORDER BY size_in_bytes DESC;
-- Result: SELECT * FROM dbo.Orders WHERE order_id = 10042
--         SELECT * FROM dbo.Orders WHERE order_id = 10043  (different literal each time)
```

**Diagnosis:** Application is sending literal-embedded SQL instead of parameterized queries. Each unique order_id creates a separate plan.

**Immediate fix (seconds to apply):**
```sql
EXEC sp_configure 'optimize for ad hoc workloads', 1;
RECONFIGURE;
-- New single-use plans store only a stub (8 bytes) instead of the full plan
-- Existing cache can be cleared carefully:
DBCC FREESYSTEMCACHE('SQL Plans');
```

**Long-term fix:** Work with the development team to parameterize queries:
```sql
-- Before (causes plan cache pollution):
-- "SELECT * FROM dbo.Orders WHERE order_id = " + orderId

-- After (shares a single cached plan):
EXEC sp_executesql N'SELECT * FROM dbo.Orders WHERE order_id = @oid',
                   N'@oid INT', @oid = 10042;
```

---

## Scenario 4: Disk I/O Issues — Backup Job Taking 3x Longer Than Usual

**Context:** Nightly full backup of `WarehouseDB` (500 GB) now takes 4.5 hours instead of the normal 1.5 hours. No changes to the server.

**Step 1 — Check I/O latency during the backup window (run while backup is active)**
```sql
SELECT DB_NAME(fs.database_id) AS database_name,
       mf.physical_name, mf.type_desc,
       fs.io_stall_read_ms / NULLIF(fs.num_of_reads, 0) AS avg_read_ms,
       fs.io_stall_write_ms / NULLIF(fs.num_of_writes, 0) AS avg_write_ms,
       fs.num_of_reads, fs.num_of_writes
FROM sys.dm_io_virtual_file_stats(NULL, NULL) fs
JOIN sys.master_files mf ON fs.database_id = mf.database_id AND fs.file_id = mf.file_id
WHERE DB_NAME(fs.database_id) = 'WarehouseDB'
ORDER BY avg_read_ms DESC;
-- Result: avg_read_ms = 182 ms (critical; SSD should be < 5 ms)
```

**Step 2 — Check disk volume free space**
```sql
SELECT DISTINCT volume_mount_point,
       total_bytes / 1024.0 / 1024 / 1024 AS total_gb,
       available_bytes / 1024.0 / 1024 / 1024 AS free_gb,
       CAST(available_bytes * 100.0 / total_bytes AS DECIMAL(5,2)) AS free_pct
FROM sys.dm_os_volume_stats(1, 1)
CROSS JOIN sys.master_files mf
CROSS APPLY sys.dm_os_volume_stats(mf.database_id, mf.file_id);
-- Result: D:\ (data volume) — 98% full, only 12 GB free
```

**Step 3 — Check current disk-intensive activities**
```sql
SELECT r.session_id, r.command, r.status,
       r.reads, r.writes,
       SUBSTRING(st.text, 1, 200) AS sql_text
FROM sys.dm_exec_requests r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) st
WHERE r.reads > 10000 OR r.writes > 10000
ORDER BY r.reads + r.writes DESC;
-- Result: A newly added ETL job is running CONCURRENTLY with the backup
```

**Diagnosis:** Two problems:
1. Disk D:\ nearly full — SSD performance degrades significantly when > 90% full.
2. An ETL job was accidentally scheduled at the same time as the backup, creating I/O contention.

**Resolution:**
1. Free space on D:\ — move old backup files off this volume immediately.
2. Reschedule the ETL job to a different time window.
3. Long-term: separate data volumes from backup volumes. Monitor disk space with alerts at 80% and 90%.

```sql
-- Verify backup compression is enabled (reduces data written to backup target)
SELECT name, value_in_use FROM sys.configurations
WHERE name = 'backup compression default';
-- Should be 1. If 0:
EXEC sp_configure 'backup compression default', 1; RECONFIGURE;
```
