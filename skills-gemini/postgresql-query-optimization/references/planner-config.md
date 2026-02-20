# Planner Configuration

PostgreSQL server parameters that influence the behavior of the cost-based query optimizer.

## Key Parameters

| Parameter | Description | Recommended Setting |
|---|---|---|
| `shared_buffers` | Memory for caching data blocks. | 25% of RAM. |
| `work_mem` | Memory for sorts, hash joins, and temporary tables. | Set according to query complexity and available RAM. |
| `maintenance_work_mem` | Memory for `VACUUM`, `CREATE INDEX`, and other maintenance tasks. | Larger for faster maintenance (e.g., 1 GB or more). |
| `effective_cache_size` | Informs the optimizer about total available cache (including OS cache). | 50-75% of RAM. |
| `random_page_cost` | Cost factor for random access vs. sequential access. | SSD: 1.1 or 1.0. HDD: 4.0. |
| `seq_page_cost` | Cost factor for sequential access. | Default: 1.0. |
| `cpu_tuple_cost` | Cost factor for processing a single row. | Default: 0.01. |
| `cpu_index_tuple_cost` | Cost factor for processing a single index row. | Default: 0.005. |
| `cpu_operator_cost` | Cost factor for processing a single operator. | Default: 0.0025. |
| `parallel_tuple_cost` | Cost factor for passing a row between processes. | Default: 0.1. |

## Tuning for Performance

1.  **Adjust `random_page_cost` for SSDs:** The default (4.0) assumes spinning disks. For SSDs, setting this closer to `seq_page_cost` (1.0 or 1.1) encourages the planner to use more indexes.
2.  **Increase `work_mem` for Complex Queries:** If your queries frequently perform large sorts or hash joins, increasing `work_mem` (e.g., to 64MB or 128MB) can keep these operations in memory, avoiding slow disk writes.
3.  **Optimize Parallelism:** Tune `max_parallel_workers_per_gather` and `max_worker_processes` to leverage multiple CPU cores for large scans and joins.
4.  **Use `effective_cache_size`:** This parameter does not allocate memory; it only informs the planner. Setting it correctly (e.g., to 75% of RAM) helps the planner accurately estimate the cost of I/O.

## References
- [PostgreSQL Documentation: Server Configuration - Query Planning](https://www.postgresql.org/docs/current/runtime-config-query.html)
- [pganalyze: Tuning PostgreSQL Query Optimizer](https://pganalyze.com/blog/postgresql-query-optimizer-cost-model)
