# Task: Build `sqlserver-health-check` Skill

## Skill Metadata

```yaml
name: sqlserver-health-check
description: Diagnoses SQL Server health by analyzing DMVs, wait statistics, blocking, memory pressure, CPU usage, disk I/O, and database integrity. Use when a SQL Server instance is slow, unresponsive, experiencing errors, or when performing routine health audits.
metadata:
  author: dba-skills
  version: "1.0"
  platform: SQL Server 2016+
```

## Directory Structure

```
sqlserver-health-check/
├── SKILL.md
├── references/
│   ├── dmv-reference.md        # Key DMVs and their purpose
│   ├── diagnostic-queries.md   # Ready-to-run T-SQL health check queries
│   └── thresholds.md           # Healthy/warning/critical thresholds
└── examples/
    └── examples.md             # Sample health check scenarios
```

## SKILL.md Content Plan

### Sections

1. **Quick Health Overview** - How to run a fast server health check
2. **Wait Statistics Analysis** - Top waits and what they mean
3. **Blocking & Deadlocks** - Identifying and resolving blocking chains
4. **Memory Pressure** - PLE, buffer pool, memory grants
5. **CPU Usage** - Top CPU-consuming queries
6. **Disk I/O** - I/O stalls, VLF count, file sizes
7. **Database Integrity** - DBCC CHECKDB, suspect pages
8. **Error Log Review** - Critical errors to look for

### Key DMVs to Cover

- `sys.dm_os_wait_stats` - Wait statistics
- `sys.dm_exec_requests` - Active requests
- `sys.dm_exec_sessions` - Active sessions
- `sys.dm_exec_query_stats` - Query stats
- `sys.dm_os_ring_buffers` - Memory notifications
- `sys.dm_io_virtual_file_stats` - I/O stats
- `sys.dm_os_performance_counters` - PLE, batch requests
- `sys.dm_tran_locks` - Lock info
- `sys.dm_exec_sql_text` - SQL text
- `sys.dm_exec_query_plan` - Execution plans
- `msdb.dbo.suspect_pages` - Suspect pages
- `sys.databases` - Database states

### Key Thresholds

| Metric | Healthy | Warning | Critical |
|---|---|---|---|
| PLE (Page Life Expectancy) | >300s | 100-300s | <100s |
| Blocking (wait time) | <5s | 5-30s | >30s |
| VLF count per DB | <50 | 50-200 | >200 |
| Log space used | <70% | 70-85% | >85% |
| CPU utilization | <70% | 70-85% | >85% |
| I/O stall (ms/op) | <20ms | 20-50ms | >50ms |

### Progressive Disclosure

- SKILL.md: Quick checks + references to detail files
- `references/dmv-reference.md`: Full DMV listing with column descriptions
- `references/diagnostic-queries.md`: Complete ready-to-run T-SQL scripts
- `references/thresholds.md`: Full threshold table with explanations
- `examples/examples.md`: 3-5 real-world health check scenarios

## References to Research

See `claude-research/sqlserver-health-check.md` for detailed findings.

## Acceptance Criteria

- [ ] Agent can run a full health check from SKILL.md instructions
- [ ] All DMV queries produce actionable output
- [ ] Thresholds are clearly stated with recommended actions
- [ ] Examples cover: slow server, blocking, memory pressure, disk issues
- [ ] No queries require undocumented features
