# Task: Build `sqlserver-query-optimization` Skill

## Skill Metadata

```yaml
name: sqlserver-query-optimization
description: Analyzes and optimizes slow SQL Server queries by reading execution plans, identifying missing indexes, detecting parameter sniffing, and resolving query regressions. Use when queries are slow, consuming excessive CPU/IO, or when investigating performance regressions after deployments or statistics updates.
metadata:
  author: dba-skills
  version: "1.0"
  platform: SQL Server 2016+
```

## Directory Structure

```
sqlserver-query-optimization/
├── SKILL.md
├── references/
│   ├── execution-plan-guide.md    # How to read and interpret execution plans
│   ├── index-strategies.md        # Covering indexes, included columns, statistics
│   └── common-antipatterns.md     # Queries patterns that hurt performance
└── examples/
    └── examples.md                # Before/after optimization scenarios
```

## SKILL.md Content Plan

### Sections

1. **Find the Slow Query** - How to identify the problematic query
2. **Read the Execution Plan** - Key operators and cost indicators
3. **Missing Index Analysis** - DMV-based missing index suggestions
4. **Parameter Sniffing** - Detection and remediation
5. **Statistics** - When to update, how to check staleness
6. **Query Rewrites** - Common problematic patterns with rewrites
7. **Query Store** - Using Query Store for regression detection

### Key DMVs and Tools

- `sys.dm_exec_query_stats` - Aggregated query performance
- `sys.dm_exec_procedure_stats` - Stored procedure performance
- `sys.dm_db_missing_index_details` - Missing index recommendations
- `sys.dm_db_missing_index_groups` - Missing index group info
- `sys.dm_db_missing_index_group_stats` - Impact of missing indexes
- `sys.dm_exec_cached_plans` - Plan cache
- `sys.dm_exec_query_plan` - XML execution plans
- `sys.dm_exec_sql_text` - Query text
- `sys.dm_db_index_usage_stats` - Index usage statistics
- Query Store: `sys.query_store_plan`, `sys.query_store_query`, `sys.query_store_runtime_stats`
- `DBCC FREEPROCCACHE` - Clear plan cache (use with caution)
- `SET STATISTICS IO ON` / `SET STATISTICS TIME ON`

### Key Concepts to Cover

- Seek vs Scan (when each is appropriate)
- Sargability (why functions on columns are bad)
- Implicit conversions and their cost
- Parameter sniffing: local variables, OPTIMIZE FOR UNKNOWN, RECOMPILE
- Statistics: auto-update thresholds, filtered statistics
- Columnstore indexes for analytical workloads
- Covering indexes and included columns
- Key lookup elimination

### Examples to Include

1. Find top 10 CPU-consuming queries
2. Find queries with missing index warnings
3. Detect parameter sniffing issue
4. Identify implicit conversion causing scan
5. Before/after: adding a covering index
6. Using Query Store to find plan regression

### Progressive Disclosure

- SKILL.md: Workflow for diagnosing a slow query + references
- `references/execution-plan-guide.md`: Detailed plan operators
- `references/index-strategies.md`: Index design patterns
- `references/common-antipatterns.md`: T-SQL anti-patterns with fixes
- `examples/examples.md`: 5 real optimization scenarios

## References to Research

See `claude-research/sqlserver-query-optimization.md` for detailed findings.

## Acceptance Criteria

- [ ] Agent can find the top slow queries without prior context
- [ ] Agent can interpret execution plan warnings and suggest fixes
- [ ] Agent can detect parameter sniffing and apply appropriate fix
- [ ] Agent can evaluate and apply missing index recommendations
- [ ] Examples show realistic query improvements
