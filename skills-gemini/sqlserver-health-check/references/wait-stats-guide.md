# Wait Stats Guide

Wait statistics are cumulative statistics stored in `sys.dm_os_wait_stats` that help identify where SQL Server is spending its time.

## Key Wait Types

| Wait Type | Description | Troubleshooting |
|---|---|---|
| `CXPACKET` | Parallelism wait. Occurs when a query is using multiple threads. | Check `MAXDOP` and `Cost Threshold for Parallelism` configuration. |
| `PAGEIOLATCH_SH` | Waiting to read a data page from disk into memory. | Check I/O latency, missing indexes, and memory pressure. |
| `PAGELATCH_UP` | Contention on internal data pages (e.g., `tempdb` PFS/GAM pages). | Add more `tempdb` data files or increase metadata caching. |
| `WRITELOG` | Waiting for transaction log write to disk to complete. | Check log file latency and transaction commit frequency. |
| `LCK_M_X` | Exclusive lock wait. Blocking is occurring. | Find the lead blocker using `sp_WhoIsActive` or `sys.dm_exec_requests`. |
| `RESOURCE_SEMAPHORE` | Memory grant wait. A query is waiting for memory to execute. | Check for queries with large memory grants and increase RAM if needed. |
| `ASYNC_NETWORK_IO` | Waiting for the client application to process data. | Ensure the client is fetching data efficiently and check network latency. |
| `HADR_SYNC_COMMIT` | (Always On AG) Waiting for a synchronous secondary to commit. | Check network latency to secondary or disk latency on the secondary. |

## Clearing Wait Stats
To start a new baseline for investigation:
```sql
DBCC SQLPERF ('sys.dm_os_wait_stats', CLEAR);
```

## References
- [SQLskills Wait Types Library](https://www.sqlskills.com/help/waits/)
- [Microsoft Documentation: sys.dm_os_wait_stats](https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-os-wait-stats-transact-sql)
