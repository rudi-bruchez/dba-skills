# SQL Server Health Check & Diagnostics Research

## Overview
SQL Server health checks are essential for maintaining performance, stability, and reliability. This research covers key metrics, Dynamic Management Views (DMVs), and best practices for 2024-2025.

## Key Performance Indicators (KPIs)
- **CPU Usage:** Monitor for sustained high CPU utilization (>80%) which can indicate inefficient queries or insufficient hardware.
- **Memory Health:**
    - **Page Life Expectancy (PLE):** Time a data page stays in the buffer pool. Low values indicate memory pressure.
    - **Buffer Cache Hit Ratio:** Percentage of pages found in memory vs. disk.
- **I/O Latency:** Check for disk read/write latencies. Anything over 20ms is typically concerning.
- **Wait Statistics:** Understand what the server is waiting for (e.g., `CXPACKET` for parallelism, `PAGEIOLATCH_SH` for disk reads).
- **Blocking & Deadlocks:** Identify sessions preventing others from proceeding.

## Essential DMVs for Health Checks
- `sys.dm_os_wait_stats`: Cumulative wait statistics across the server.
- `sys.dm_exec_requests`: Currently executing requests, resource consumption, and wait states.
- `sys.dm_os_performance_counters`: Access to SQL Server performance counters (PLE, Cache Hit Ratio).
- `sys.dm_exec_query_stats`: Aggregate performance statistics for cached query plans.
- `sys.dm_io_virtual_file_stats`: File-level I/O latency and throughput.
- `sys.dm_os_memory_clerks`: Memory allocation by internal components.
- `sys.dm_db_index_usage_stats`: Usage statistics for indexes (useful for finding unused indexes).

## Diagnostic Tools & Procedures
- **sp_WhoIsActive:** A popular community stored procedure for real-time monitoring of active sessions.
- **sp_Blitz / sp_BlitzFirst:** Community tools for rapid health assessment and performance monitoring.
- **Query Store:** Captures historical query performance data and execution plans.
- **Extended Events:** Low-overhead tracing for capturing specific events (e.g., deadlocks, long-running queries).

## Best Practices (2024-2025)
- **Baseline Establishment:** Regularly capture metrics to understand "normal" behavior.
- **Automated Alerts:** Set up alerts for critical thresholds (e.g., low disk space, high CPU).
- **Regular Maintenance:** Ensure statistics are updated and indexes are defragmented/reorganized.
- **Patch Management:** Keep SQL Server updated with the latest Cumulative Updates (CUs).
- **SQL Server 2025 New Features:** Utilize `sys.dm_os_memory_health_history` for easier memory pressure troubleshooting.

## References
- [Microsoft Documentation on DMVs](https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/system-dynamic-management-views)
- [Brent Ozar's SQL Server Health Check Guide](https://www.brentozar.com/sql-server-health-check/)
- [Glenn Berry's Diagnostic Queries](https://glennsqlperformance.com/resources/)
