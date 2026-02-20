# Task: Build `sqlserver-index-management` Skill

## Skill Metadata

```yaml
name: sqlserver-index-management
description: Manages SQL Server indexes including analysis of missing indexes, fragmentation assessment, maintenance scheduling, and index design recommendations. Use when diagnosing slow queries due to missing indexes, planning index maintenance windows, removing unused indexes, or designing index strategies for new tables.
metadata:
  author: dba-skills
  version: "1.0"
  platform: SQL Server 2016+
```

## Directory Structure

```
sqlserver-index-management/
├── SKILL.md
├── references/
│   ├── index-types.md           # B-tree, columnstore, filtered, spatial indexes
│   ├── maintenance-scripts.md   # Fragmentation check and rebuild/reorganize scripts
│   └── index-design.md          # Index design patterns and best practices
└── examples/
    └── examples.md              # Index analysis and maintenance scenarios
```

## SKILL.md Content Plan

### Sections

1. **Index Health Check** - Assess overall index health quickly
2. **Missing Index Analysis** - Find and evaluate missing index recommendations
3. **Unused Index Detection** - Find indexes that waste resources
4. **Fragmentation Analysis** - When to rebuild vs reorganize
5. **Maintenance Strategy** - Scheduling maintenance for large databases
6. **Index Design Principles** - Key guidelines for new index creation
7. **Columnstore Indexes** - When and how to use them

### Key DMVs and Views

- `sys.dm_db_missing_index_details` - Missing index details
- `sys.dm_db_missing_index_groups` - Missing index groups
- `sys.dm_db_missing_index_group_stats` - Missing index impact scores
- `sys.dm_db_index_physical_stats` - Fragmentation statistics
- `sys.dm_db_index_usage_stats` - Seeks, scans, lookups, updates
- `sys.indexes` - Index metadata
- `sys.index_columns` - Index column definitions
- `sys.stats` - Statistics objects
- `sys.stats_columns` - Statistics columns
- `DBCC SHOW_STATISTICS` - Statistics histogram
- `sys.dm_exec_query_stats` - Query plan stats

### Key T-SQL Operations

- `ALTER INDEX ... REBUILD` - Full index rebuild (updates stats)
- `ALTER INDEX ... REORGANIZE` - Online defrag (partial fix)
- `ALTER INDEX ... DISABLE` - Disable without dropping
- `UPDATE STATISTICS` - Manual statistics update
- `CREATE INDEX ... WITH (ONLINE = ON)` - Online index build
- `CREATE COLUMNSTORE INDEX` - Columnstore creation
- `sp_updatestats` - Update all statistics

### Index Maintenance Thresholds

| Fragmentation | Action |
|---|---|
| < 5% | No action needed |
| 5–30% | REORGANIZE |
| > 30% | REBUILD |
| Very small tables | Skip (fragmentation less meaningful) |

### Missing Index Score Formula

```
Impact score = avg_user_impact * (user_seeks + user_scans)
```
Prioritize indexes with score > 10,000 that have high seek counts.

### Progressive Disclosure

- SKILL.md: Quick health check + maintenance workflow + links
- `references/index-types.md`: All index types with use cases
- `references/maintenance-scripts.md`: Complete maintenance T-SQL scripts
- `references/index-design.md`: Design patterns and anti-patterns
- `examples/examples.md`: Missing index evaluation, removing duplicates

## References to Research

See `claude-research/sqlserver-index-management.md` for detailed findings.

## Acceptance Criteria

- [ ] Agent can assess full index health for a database
- [ ] Agent can prioritize missing index recommendations
- [ ] Agent can generate appropriate maintenance scripts
- [ ] Agent can identify and safely drop unused indexes
- [ ] Agent can design indexes for a given query workload
