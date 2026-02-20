# Task: Build `postgresql-health-check` Skill

## Skill Metadata

```yaml
name: postgresql-health-check
description: Diagnoses PostgreSQL instance health by analyzing wait events, connection usage, bloat, autovacuum status, lock contention, and replication lag. Use when a PostgreSQL server is slow, connections are being refused, tables are bloated, autovacuum is not keeping up, or during routine health audits.
metadata:
  author: dba-skills
  version: "1.0"
  platform: PostgreSQL 13+
```

## Directory Structure

```
postgresql-health-check/
├── SKILL.md
├── references/
│   ├── pg-stat-reference.md     # Key pg_stat_* views and their columns
│   ├── diagnostic-queries.md    # Ready-to-run diagnostic SQL queries
│   └── thresholds.md            # Healthy/warning/critical thresholds
└── examples/
    └── examples.md              # Health check scenarios
```

## SKILL.md Content Plan

### Sections

1. **Quick Health Overview** - Fast server health check workflow
2. **Connection & Activity** - Connection pool saturation, idle connections
3. **Wait Events & Lock Contention** - Blocking queries and lock waits
4. **Vacuum & Bloat** - Autovacuum status, table/index bloat
5. **Replication Lag** - Streaming replication monitoring
6. **Cache Hit Rate** - Buffer cache effectiveness
7. **Long-Running Queries** - Finding and managing long queries
8. **Checkpoint & BGWriter** - WAL write performance

### Key System Views

- `pg_stat_activity` - Current connections and queries
- `pg_stat_database` - Per-database statistics (cache hits, deadlocks)
- `pg_stat_user_tables` - Table-level statistics (seq scans, dead tuples)
- `pg_stat_user_indexes` - Index usage statistics
- `pg_statio_user_tables` - Table I/O statistics
- `pg_stat_bgwriter` - Background writer statistics
- `pg_stat_replication` - Streaming replication status
- `pg_stat_wal_receiver` - WAL receiver status (on standby)
- `pg_locks` - Lock information
- `pg_blocking_pids()` - Find blocking process IDs
- `pg_stat_progress_vacuum` - Vacuum progress
- `pg_stat_progress_analyze` - Analyze progress
- `pg_size_pretty()`, `pg_total_relation_size()` - Size utilities
- `pg_database_size()` - Database size

### Key Health Metrics

| Metric | Healthy | Warning | Critical |
|---|---|---|---|
| Connection usage | < 70% of max_connections | 70–85% | > 85% |
| Cache hit ratio | > 99% | 95–99% | < 95% |
| Dead tuple ratio | < 5% | 5–20% | > 20% |
| Replication lag | < 10 MB | 10–100 MB | > 100 MB |
| Long-running query | < 1 min | 1–5 min | > 5 min |
| Transaction age (wraparound) | < 500M xids | > 1B xids | > 1.5B xids (DANGER) |

### Key Concerns Unique to PostgreSQL

- Transaction ID wraparound (autovacuum must run)
- Table bloat from dead tuples (MVCC)
- Index bloat after many updates/deletes
- Connection exhaustion (no native connection pooling)
- Autovacuum being blocked by long transactions

### Progressive Disclosure

- SKILL.md: Quick health check + top issues + links
- `references/pg-stat-reference.md`: Full pg_stat_* view documentation
- `references/diagnostic-queries.md`: Complete diagnostic SQL library
- `references/thresholds.md`: Full threshold table with remediation steps
- `examples/examples.md`: Bloat, connection exhaustion, wraparound scenarios

## References to Research

See `claude-research/postgresql-health-check.md` for detailed findings.

## Acceptance Criteria

- [ ] Agent can run a comprehensive health check
- [ ] Agent can identify transaction wraparound risk
- [ ] Agent can find and report on table/index bloat
- [ ] Agent can diagnose connection exhaustion
- [ ] Examples cover: bloat, wraparound risk, replication lag, blocking
