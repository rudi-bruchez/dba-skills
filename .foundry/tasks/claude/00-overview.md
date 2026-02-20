# DBA Skills - Build Plan Overview

This document outlines the complete plan for building AI agent skills for database administration.

## Skills to Build

### SQL Server (6 skills)

| Skill Name | Directory | Description |
|---|---|---|
| `sqlserver-health-check` | `sqlserver-health-check/` | Diagnose SQL Server health via DMVs |
| `sqlserver-query-optimization` | `sqlserver-query-optimization/` | Analyze and optimize slow queries |
| `sqlserver-backup-recovery` | `sqlserver-backup-recovery/` | Backup strategies and restore operations |
| `sqlserver-security` | `sqlserver-security/` | Security audit and hardening |
| `sqlserver-index-management` | `sqlserver-index-management/` | Index analysis, creation, and maintenance |
| `sqlserver-high-availability` | `sqlserver-high-availability/` | Always On, log shipping, HA/DR |

### PostgreSQL (5 skills)

| Skill Name | Directory | Description |
|---|---|---|
| `postgresql-health-check` | `postgresql-health-check/` | Diagnose PostgreSQL health via pg_stat views |
| `postgresql-query-optimization` | `postgresql-query-optimization/` | EXPLAIN plans, indexes, vacuuming |
| `postgresql-backup-recovery` | `postgresql-backup-recovery/` | pg_dump, pg_basebackup, PITR |
| `postgresql-security` | `postgresql-security/` | Roles, pg_hba.conf, SSL, row security |
| `postgresql-replication` | `postgresql-replication/` | Streaming replication, Patroni, pgBouncer |

## Skill Structure (per agentskills.io spec)

Each skill directory follows this layout:

```
skill-name/
├── SKILL.md                    # Required: metadata + main instructions (<500 lines)
├── references/
│   ├── REFERENCE.md            # Detailed technical reference
│   ├── diagnostic-queries.md   # Ready-to-use diagnostic SQL
│   └── best-practices.md       # Best practices checklist
└── examples/
    └── examples.md             # Concrete input/output examples
```

## SKILL.md Requirements (per spec)

- YAML frontmatter: `name`, `description` (required), `metadata` (optional)
- `name`: max 64 chars, lowercase + hyphens only
- `description`: max 1024 chars, third person, includes what + when to use
- Body: under 500 lines, use progressive disclosure
- Reference detailed content to `references/` files

## Content Standards

- All SQL queries must be tested/validated
- Include specific DMV/system view names
- Include thresholds and numeric benchmarks where applicable
- Provide concrete examples (not abstract descriptions)
- Use consistent terminology within each skill
- No time-sensitive information (use "current" patterns, not date references)
- Forward slashes in all file paths

## Research Sources

See `claude-research/` directory for detailed research per topic.

## Build Order

1. SQL Server skills (more widely used in enterprise context)
2. PostgreSQL skills
3. Cross-validate for consistency
