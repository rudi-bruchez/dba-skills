# SQL Server Health Metric Thresholds

Reference thresholds for common SQL Server health indicators. Thresholds are guidelines — establish a baseline for each environment and alert on deviations from baseline, not only on absolute values.

---

## Memory Thresholds

| Metric | Query | Healthy | Warning | Critical | Remediation |
|---|---|---|---|---|---|
| Page Life Expectancy (PLE) | `sys.dm_os_performance_counters` | > 1000 sec | 300–1000 sec | < 300 sec | Increase max server memory; add RAM; fix high-read queries |
| Single-use plan cache | `sys.dm_exec_cached_plans WHERE usecounts=1` | < 100 MB | 100–500 MB | > 500 MB | Enable `optimize for ad hoc workloads` |
| Process physical memory low | `sys.dm_os_process_memory` | 0 (false) | — | 1 (true) | Reduce max server memory; add RAM |
| Buffer pool % of RAM | `sys.dm_os_memory_clerks` | 70–90% | < 50% | < 30% | Verify max server memory is set correctly |

**PLE Formula for systems with > 16 GB RAM:**
PLE benchmark = (RAM in GB / 4 GB) × 300. For a 64 GB server: 64/4 × 300 = 4800. Use this as the "healthy" target rather than the legacy 300 value.

---

## I/O Latency Thresholds

| Storage Type | Metric | Healthy | Warning | Critical |
|---|---|---|---|---|
| SSD / NVMe — Read | `avg_read_ms` from `sys.dm_io_virtual_file_stats` | < 5 ms | 5–10 ms | > 10 ms |
| SSD / NVMe — Write | `avg_write_ms` | < 2 ms | 2–5 ms | > 5 ms |
| HDD — Read | `avg_read_ms` | < 20 ms | 20–30 ms | > 30 ms |
| HDD — Write | `avg_write_ms` | < 10 ms | 10–20 ms | > 20 ms |
| Log file write | `avg_write_ms` for type_desc = 'LOG' | < 2 ms | 2–10 ms | > 10 ms |

**Query:**
```sql
SELECT
    DB_NAME(fs.database_id)                                               AS database_name,
    mf.physical_name,
    mf.type_desc,
    fs.io_stall_read_ms  / NULLIF(fs.num_of_reads, 0)                    AS avg_read_ms,
    fs.io_stall_write_ms / NULLIF(fs.num_of_writes, 0)                   AS avg_write_ms
FROM sys.dm_io_virtual_file_stats(NULL, NULL) fs
JOIN sys.master_files mf ON fs.database_id = mf.database_id AND fs.file_id = mf.file_id
ORDER BY avg_write_ms DESC;
```

---

## CPU Thresholds

| Metric | Healthy | Warning | Critical | Remediation |
|---|---|---|---|---|
| SQL Server CPU % (sustained) | < 70% | 70–85% | > 85% | Optimize top CPU queries; add cores |
| Signal wait % of total waits | < 25% | 25–50% | > 50% | CPU pressure — optimize queries; review MAXDOP |
| Scheduler runnable queue (`runnable_tasks_count`) | 0–1 per scheduler | 2–4 | > 4 | CPU bound; reduce concurrency |
| Compilations/sec % of batch requests | < 10% | 10–20% | > 20% | Parameterize queries; use stored procedures |
| Re-compilations/sec | < 5% of compilations | 5–15% | > 15% | Investigate SET options changing; use temp tables cautiously |

---

## Blocking and Locking Thresholds

| Metric | Healthy | Warning | Critical | Remediation |
|---|---|---|---|---|
| Blocked sessions count | 0 | 1–5 | > 5 | Identify and resolve head blocker; optimize queries |
| Blocking duration | < 5 sec | 5–30 sec | > 30 sec | Kill head blocker if critical; add indexes; enable RCSI |
| Deadlocks per hour | 0 | 1–5 | > 5 | Add/reorder indexes; keep transactions short; use RCSI |
| Lock wait time avg | < 100 ms | 100–500 ms | > 500 ms | Add indexes on join/filter columns |

**Alert query:**
```sql
SELECT COUNT(*) AS blocked_count
FROM sys.dm_exec_requests
WHERE blocking_session_id > 0;
```

---

## Transaction Log Thresholds

| Metric | Healthy | Warning | Critical | Remediation |
|---|---|---|---|---|
| Log space used % | < 60% | 60–80% | > 80% | Take log backup (FULL recovery); check `log_reuse_wait_desc` |
| VLF count | < 200 | 200–1000 | > 1000 | Shrink log to small size, then grow with fixed increment (e.g., 512 MB) |
| `log_reuse_wait_desc` = 'LOG_BACKUP' | N/A — take log backup immediately | | | `BACKUP LOG [db] TO DISK = '...'` |
| `log_reuse_wait_desc` = 'ACTIVE_TRANSACTION' | N/A | | | Find and commit/rollback the long-running transaction |

```sql
-- Log space by database
SELECT DB_NAME() AS database_name,
       used_log_space_percent,
       log_backup_time
FROM sys.dm_db_log_space_usage;

-- VLF count
SELECT DB_NAME(database_id) AS database_name, COUNT(*) AS vlf_count
FROM sys.dm_db_log_info(NULL)
GROUP BY database_id
ORDER BY vlf_count DESC;
```

---

## Database State Thresholds

| Setting | Expected Value | Problem Value | Fix |
|---|---|---|---|
| Database state | ONLINE | SUSPECT, EMERGENCY, RESTORING | Investigate; may require DBCC CHECKDB or restore |
| `is_auto_shrink_on` | 0 (OFF) | 1 (ON) | `ALTER DATABASE [db] SET AUTO_SHRINK OFF` — auto_shrink causes index fragmentation |
| `is_auto_close_on` | 0 (OFF) | 1 (ON) | `ALTER DATABASE [db] SET AUTO_CLOSE OFF` — causes overhead on every connection |
| `page_verify_option_desc` | CHECKSUM | TORN_PAGE_DETECTION or NONE | `ALTER DATABASE [db] SET PAGE_VERIFY CHECKSUM` |
| `is_auto_update_stats_on` | 1 (ON) | 0 (OFF) | Enable unless stats are managed manually |
| `compatibility_level` | Current SQL version level | 2+ versions behind | Test and upgrade compatibility level |

---

## Backup Monitoring Thresholds

| Metric | Alert Threshold | Query |
|---|---|---|
| Last full backup age | > 25 hours | `msdb.dbo.backupset` WHERE type='D' |
| Last log backup age (FULL recovery) | > 60 minutes | `msdb.dbo.backupset` WHERE type='L' |
| Backup duration increase | > 20% week-over-week | Track in `msdb.dbo.backupset` |
| Backup size growth | > 10% per week | Monitor compressed_backup_size trend |

---

## Configuration Best Practice Values

| Setting | Recommended Value | Default | Query |
|---|---|---|---|
| max server memory | Physical RAM - 2–4 GB for OS | 2,147,483,647 (unlimited) | `sys.configurations` |
| max degree of parallelism | Depends: 4–8 for OLTP; 0 for DW | 0 | `sys.configurations` |
| cost threshold for parallelism | 25–50 | 5 (too low) | `sys.configurations` |
| optimize for ad hoc workloads | 1 (ON) | 0 (OFF) | `sys.configurations` |
| backup compression default | 1 (ON) | 0 (OFF) | `sys.configurations` |
| xp_cmdshell | 0 (OFF) | 0 (OFF) | `sys.configurations` |
