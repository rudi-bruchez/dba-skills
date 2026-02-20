# Thresholds & Key Performance Indicators (KPIs)

Use these benchmarks to assess the health of your SQL Server instance.

## Metrics at a Glance

| Metric | Target / Benchmark | Warning Threshold | Critical Threshold |
|---|---|---|---|
| **CPU Utilization** | < 60% average | > 80% sustained | > 90% sustained |
| **I/O Latency (Data)** | < 10ms | > 20ms | > 50ms |
| **I/O Latency (Log)** | < 5ms | > 10ms | > 20ms |
| **Page Life Expectancy** | > 300s (per 4GB buffer) | < 300s | < 100s |
| **Buffer Cache Hit Ratio**| > 98% (OLTP) | < 95% | < 90% |
| **Wait Stats (%)** | Individual < 20% | Individual > 30% | Individual > 50% |
| **Deadlocks / Minute** | 0 | > 1 | > 5 |
| **Blocking / Minute** | < 10% of sessions | > 20% | > 40% |

## Interpretation Guide

- **High CPU + High CXPACKET:** Potential over-parallelization or missing indexes.
- **Low PLE + High PAGEIOLATCH_SH:** Memory pressure. The buffer pool is constantly flushing and reading from disk.
- **High WRITELOG:** Slow log file disks or too many small, frequent commits.
- **High RESOURCE_SEMAPHORE:** Query workload needs more RAM or optimized memory grants.
- **High ASYNC_NETWORK_IO:** The application is requesting more data than it can process quickly (e.g., large `SELECT *` without filtering).

## 2024-2025 Considerations

- **Modern I/O:** For NVMe storage, I/O latency should ideally be < 1ms.
- **Always On AG:** Synchronous replicas can cause `HADR_SYNC_COMMIT` waits; ensure network bandwidth is sufficient.
- **Cloud Databases:** Azure SQL Database and AWS RDS have their own specific DTU/vCore monitoring metrics.
