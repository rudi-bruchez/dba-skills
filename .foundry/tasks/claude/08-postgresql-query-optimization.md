# Task: Build `postgresql-query-optimization` Skill

## Skill Metadata

```yaml
name: postgresql-query-optimization
description: Analyzes and optimizes slow PostgreSQL queries by interpreting EXPLAIN ANALYZE output, identifying missing indexes, resolving sequential scans, and tuning planner configuration. Use when queries are slow, the query planner chooses bad plans, pg_stat_statements shows expensive queries, or after schema changes cause performance regressions.
metadata:
  author: dba-skills
  version: "1.0"
  platform: PostgreSQL 13+
```

## Directory Structure

```
postgresql-query-optimization/
├── SKILL.md
├── references/
│   ├── explain-guide.md         # How to read EXPLAIN ANALYZE output
│   ├── index-types.md           # B-tree, Hash, GIN, GiST, BRIN indexes
│   └── planner-config.md        # Planner cost parameters and statistics
└── examples/
    └── examples.md              # Before/after query optimization scenarios
```

## SKILL.md Content Plan

### Sections

1. **Find Slow Queries** - Using pg_stat_statements and pg_stat_activity
2. **Reading EXPLAIN ANALYZE** - Key nodes and what they indicate
3. **Index Analysis** - Which indexes to create and what type
4. **Statistics & Planner** - Table statistics, planner cost estimation
5. **Common Query Rewrites** - Patterns that help the planner
6. **Partitioning** - When table partitioning helps performance
7. **Configuration Tuning** - Key planner parameters

### Key Tools and Views

- `EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)` - Execution plan with actual costs
- `pg_stat_statements` - Aggregated query stats (requires extension)
- `pg_stat_activity` - Currently running queries
- `pg_stat_user_tables` - Sequential scan counts
- `pg_stat_user_indexes` - Index usage counts
- `pg_indexes` - Index definitions
- `\d+ tablename` (psql) - Table and index info
- `ANALYZE tablename` - Update table statistics
- `VACUUM ANALYZE` - Vacuum + update statistics
- `pg_stats` - Column statistics
- `pg_class.reltuples` - Estimated row count

### EXPLAIN ANALYZE Key Nodes

| Node Type | Meaning | When Bad |
|---|---|---|
| Seq Scan | Full table scan | Large table, no index |
| Index Scan | B-tree index lookup | Generally good |
| Index Only Scan | Heap not accessed | Best case |
| Bitmap Heap Scan | Multiple row fetch | OK for large result sets |
| Hash Join | In-memory hash build | When hash too large |
| Nested Loop | Row-by-row join | N+1 problem on large sets |
| Merge Join | Sort-based join | Generally efficient |
| Sort | Explicit sort | Can be expensive |

### Index Type Selection Guide

| Data Type / Query Pattern | Index Type |
|---|---|
| Equality, range, ORDER BY | B-tree (default) |
| Equality only (int, uuid) | Hash |
| Array containment, JSONB | GIN |
| Geometric, full-text | GiST |
| Large tables, date ranges (sequential) | BRIN |
| Partial data (filtered) | Partial index |
| Multiple columns | Composite B-tree |

### Configuration Parameters to Tune

- `work_mem` - Per-sort/hash operation memory
- `random_page_cost` - Cost of random disk access (lower for SSD)
- `seq_page_cost` - Cost of sequential disk read
- `effective_cache_size` - OS cache hint for planner
- `default_statistics_target` - Default statistics granularity
- `enable_seqscan`, `enable_hashjoin` - Force plan choices (debug only)
- `jit` - JIT compilation (can help or hurt)

### Progressive Disclosure

- SKILL.md: Find slow queries + read EXPLAIN + links to details
- `references/explain-guide.md`: EXPLAIN output interpretation guide
- `references/index-types.md`: All index types with when to use each
- `references/planner-config.md`: Configuration parameters and tuning
- `examples/examples.md`: 5 optimization scenarios

## References to Research

See `claude-research/postgresql-query-optimization.md` for detailed findings.

## Acceptance Criteria

- [ ] Agent can find and rank slow queries using pg_stat_statements
- [ ] Agent can read and explain EXPLAIN ANALYZE output
- [ ] Agent can recommend the right index type for a query
- [ ] Agent can identify and fix sequential scans on large tables
- [ ] Agent can tune planner parameters for specific workloads
