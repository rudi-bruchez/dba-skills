# Thresholds & Key Performance Indicators (KPIs)

Use these benchmarks to assess the health of your PostgreSQL instance.

## Metrics at a Glance

| Metric | Target / Benchmark | Warning Threshold | Critical Threshold |
|---|---|---|---|
| **Active Connections** | < 50% of `max_connections` | > 80% | > 95% |
| **Cache Hit Ratio** | > 95% (OLTP) | < 90% | < 80% |
| **Rollback Rate (%)** | < 1% | > 5% | > 10% |
| **Dead Tuples (%)** | < 5% | > 10% | > 20% |
| **Deadlocks / Minute** | 0 | > 1 | > 5 |
| **I/O Wait Events (%)** | < 10% of activity | > 20% | > 40% |
| **Index Usage (%)** | > 90% (for busy tables) | < 70% | < 50% |
| **Replication Lag** | < 10 MB or < 1s | > 100 MB | > 1 GB |

## Interpretation Guide

- **High Active Connections + High I/O Waits:** Potential disk bottleneck or missing indexes causing slow queries.
- **Low Cache Hit Ratio:** `shared_buffers` might be too small, or the working set is much larger than RAM.
- **High Dead Tuples:** Autovacuum is not keeping up or is being blocked by long-running transactions.
- **High Rollback Rate:** Application errors or logic issues causing frequent transaction failures.
- **Low Index Usage:** Queries are performing sequential scans instead of using indexes. Check `EXPLAIN ANALYZE` for top tables.

## 2024-2025 Considerations

- **Modern I/O:** For NVMe storage, even with many concurrent reads, latency should remain low.
- **Connection Saturation:** If you frequently hit `max_connections`, implement a connection pooler like `pgBouncer`.
- **Large Tables:** For tables > 100 GB, monitor bloat and autovacuum duration more closely.
- **Managed Services:** (RDS, Cloud SQL) monitor specific cloud metrics like Burst Balance and Provisioned IOPS.
