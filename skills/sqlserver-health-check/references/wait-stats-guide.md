# SQL Server Wait Statistics Guide

A wait type represents what SQL Server threads are waiting for. The top wait types identify the root cause of slowdowns better than any other single metric.

---

## How to Use This Guide

1. Run the top wait stats query from `diagnostic-queries.md`.
2. Find the dominant wait type(s) in the list below.
3. Follow the remediation steps.
4. Re-run after changes to confirm improvement.

**Note:** Always exclude benign/idle wait types from analysis. The query in `diagnostic-queries.md` already handles this exclusion.

---

## I/O-Related Waits

### PAGEIOLATCH_SH / PAGEIOLATCH_EX / PAGEIOLATCH_UP
- **Meaning:** Thread waiting for a data page to be read from disk into the buffer pool.
- **Root cause:** Missing indexes causing large scans; insufficient RAM (buffer pool too small); slow disk I/O.
- **Diagnosis:** Check `sys.dm_io_virtual_file_stats` for high read latency. Look for missing index recommendations. Check PLE.
- **Remediation:** Add covering indexes to reduce I/O. Increase max server memory. Move data files to faster storage (NVMe/SSD). Add RAM.

### WRITELOG
- **Meaning:** Thread waiting for the transaction log to be flushed to disk.
- **Root cause:** Transaction log I/O is the bottleneck. Log drive too slow, or transactions committing too frequently.
- **Diagnosis:** Check `sys.dm_io_virtual_file_stats` for log file write latency > 2 ms (SSD) or > 10 ms (HDD).
- **Remediation:** Move log file to dedicated fast disk (NVMe preferred). Reduce transaction batch size (fewer, larger commits). Check VLF count.

### IO_COMPLETION
- **Meaning:** Waiting for non-data I/O operations (backups, restore, sort spills).
- **Root cause:** Backup running to slow disk; sort spills to TempDB.
- **Remediation:** Move backups to faster destination. Fix queries causing TempDB spills (add indexes, update statistics).

### ASYNC_IO_COMPLETION
- **Meaning:** Waiting for asynchronous I/O to complete (often backup/restore operations).
- **Remediation:** Investigate active backup/restore jobs. Ensure backup media is fast enough.

---

## CPU-Related Waits

### SOS_SCHEDULER_YIELD
- **Meaning:** A worker thread voluntarily yielded the CPU scheduler to allow other threads to run, then waited to be rescheduled.
- **Root cause:** Heavy CPU usage; threads cannot get enough CPU time. Also associated with spinlock contention.
- **Diagnosis:** Check CPU utilization via ring buffer query. High `SOS_SCHEDULER_YIELD` + high CPU% = CPU bound.
- **Remediation:** Optimize CPU-intensive queries. Reduce MAXDOP for parallel queries that are competing for threads. Add CPU cores.

### CXPACKET / CXCONSUMER
- **Meaning:** Parallel query execution — one thread waiting for others to produce (CXPACKET) or consume (CXCONSUMER) data.
- **Root cause:** Parallelism imbalance; skewed data distribution among parallel threads. Not always a problem — some parallelism waits are normal.
- **Diagnosis:** Check MAXDOP setting. Look at `cost threshold for parallelism` (default 5 is too low — set to 25–50). Review execution plans for parallel operators on small result sets.
- **Remediation:** Increase `cost threshold for parallelism` (25–50). Set appropriate MAXDOP. Add query hint `OPTION (MAXDOP 1)` to specific problematic queries.

### THREADPOOL
- **Meaning:** No worker thread available to run the request. Thread pool exhausted.
- **Root cause:** Too many concurrent connections, each requiring a thread. Often seen with blocking chains where threads pile up.
- **Diagnosis:** Check `sys.dm_os_schedulers` for high `active_workers_count`. Check for blocking chains.
- **Remediation:** Resolve blocking. Use connection pooling. Consider `max worker threads` configuration (rarely needed — resolve root cause first).

---

## Locking and Blocking Waits

### LCK_M_S / LCK_M_U / LCK_M_X / LCK_M_SCH_S
- **Meaning:** Thread waiting to acquire a shared (S), update (U), exclusive (X), or schema stability lock.
- **Root cause:** Blocking — another session holds an incompatible lock. Long-running transactions; missing indexes causing lock escalation.
- **Diagnosis:** Check `sys.dm_exec_requests` for `blocking_session_id > 0`. Find the head blocker.
- **Remediation:** Add indexes on join/filter columns to reduce lock duration. Shorten transactions (commit sooner). Enable Read Committed Snapshot Isolation (RCSI): `ALTER DATABASE [YourDB] SET READ_COMMITTED_SNAPSHOT ON`. Resolve head blocker.

### LCK_M_IS / LCK_M_IU / LCK_M_IX
- **Meaning:** Intent lock waits — less severe than full exclusive locks but still blocking.
- **Root cause:** Same as LCK_M_* above.

---

## Memory-Related Waits

### RESOURCE_SEMAPHORE
- **Meaning:** Query waiting for a memory grant to execute (e.g., to allocate sort/hash buffers).
- **Root cause:** Insufficient memory for the query workload. Too many concurrent memory-intensive queries. Incorrect cardinality estimates causing oversized grants.
- **Diagnosis:** Check PLE. Check `sys.dm_exec_query_resource_semaphores` for pending grants.
- **Remediation:** Update statistics (to fix cardinality). Add indexes (reduce sort/hash requirements). Increase `max server memory`. Fix queries requesting too much memory.

### RESOURCE_SEMAPHORE_QUERY_COMPILE
- **Meaning:** Too many concurrent query compilations. Compilation requires memory from a separate semaphore.
- **Root cause:** High compilation rate — often caused by unparameterized ad hoc SQL.
- **Remediation:** Enable `optimize for ad hoc workloads`. Use parameterized queries / stored procedures. Enable Query Store.

### CMEMTHREAD
- **Meaning:** Thread waiting for a memory object to become available.
- **Root cause:** Contention on concurrent memory allocations — often plan cache or memory clerk operations.
- **Remediation:** Enable `optimize for ad hoc workloads`. Check for single-use plan cache pollution.

---

## Log and Transaction Waits

### HADR_SYNC_COMMIT
- **Meaning:** Primary replica waiting for Always On synchronous secondary to acknowledge log hardening before committing.
- **Root cause:** Network latency to secondary; secondary disk I/O slow.
- **Diagnosis:** Check `sys.dm_hadr_database_replica_states` for `log_send_queue_size` and `redo_queue_size`.
- **Remediation:** Reduce network round-trip time. Upgrade secondary disk. Consider moving some replicas to async mode.

### HADR_WORK_QUEUE (benign, usually filtered)
- **Meaning:** Always On background worker thread waiting for work.
- **Action:** Normal; no action needed.

### LOG_RATE_GOVERNOR
- **Meaning:** SQL Server throttling log generation rate (Azure SQL Database/Managed Instance).
- **Remediation:** Reduce write transaction volume. Optimize bulk insert patterns.

---

## Network and Client Waits

### ASYNC_NETWORK_IO
- **Meaning:** SQL Server has results ready, but the client is not reading them fast enough.
- **Root cause:** Slow client application; network bottleneck between SQL Server and client; large result sets not consumed promptly.
- **Diagnosis:** High `ASYNC_NETWORK_IO` with specific sessions — check the application consuming those sessions.
- **Remediation:** Ensure applications consume result sets promptly. Reduce result set size (pagination). Use SET ROWCOUNT or TOP in queries.

---

## TempDB Waits

### PAGELATCH_UP / PAGELATCH_EX on TempDB
- **Meaning:** Page latch contention on TempDB allocation pages (GAM, SGAM, PFS).
- **Root cause:** TempDB allocation contention — too many objects being created/dropped concurrently in TempDB.
- **Diagnosis:** Check if contention is on pages 1, 2, or 3 in TempDB (allocation bitmaps).
- **Remediation:** Increase TempDB data files to match CPU core count (up to 8). Pre-create temp tables. Use table variables for small datasets. Enable TempDB metadata optimization (SQL 2019+).

---

## Miscellaneous Waits

### OLEDB
- **Meaning:** Waiting for a linked server or OLEDB provider to return results.
- **Root cause:** Linked server query is slow; remote server overloaded.
- **Remediation:** Optimize the linked server query. Consider ETL approach instead of linked server live queries.

### DBMIRROR_EVENTS_QUEUE (usually benign)
- **Meaning:** Database mirroring background thread waiting for events.
- **Action:** Normal if mirroring is in use; otherwise check if mirroring endpoint is active.

### WAIT_XTP_OFFLINE_CKPT_NEW_LOG (benign)
- **Meaning:** In-Memory OLTP background checkpoint thread.
- **Action:** Normal.
