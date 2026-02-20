# Codex DBA Skills Portfolio and Standards

## Objective
Build a Codex-specific DBA skill suite for SQL Server and PostgreSQL aligned with:
- Repository intent in `README.md`
- Agent Skills Specification at https://agentskills.io/specification
- 2024-2025 operational best practices from official vendor documentation

## Planned Skill Portfolio
1. `sqlserver-health-check`
2. `sqlserver-query-optimization`
3. `sqlserver-backup-recovery`
4. `sqlserver-security`
5. `sqlserver-index-maintenance`
6. `sqlserver-high-availability`
7. `postgresql-health-check`
8. `postgresql-query-optimization`
9. `postgresql-backup-recovery`
10. `postgresql-security`
11. `postgresql-replication`

## Spec-Critical Constraints
- Skill directory contains `SKILL.md` with YAML frontmatter.
- Frontmatter includes only:
  - `name`
  - `description`
- `description` must include trigger contexts.
- `SKILL.md` body should stay concise and operational.
- Deep domain content belongs in `references/`.
- Practical scenarios belong in `examples/`.

## Shared Engineering Standards
- Use lowercase, hyphenated names under 64 chars.
- Split SQL Server and PostgreSQL guidance explicitly.
- Default to read-only diagnostics first.
- Include risk tier (`low`, `medium`, `high`) for every proposed action.
- Include rollback or mitigation for any risky operation.
- Output evidence + recommendation, not recommendation only.

## Core Sources
- Agent Skills Specification: https://agentskills.io/specification
- SQL Server Query Store: https://learn.microsoft.com/en-us/sql/relational-databases/performance/query-store-usage-scenarios?view=sql-server-ver17
- SQL Server Always On AG: https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/overview-of-always-on-availability-groups-sql-server?view=sql-server-ver17
- SQL Server backup checksums: https://learn.microsoft.com/en-us/sql/relational-databases/backup-restore/possible-media-errors-during-backup-and-restore-sql-server?view=sql-server-ver17
- SQL Server CHECKDB: https://learn.microsoft.com/en-us/sql/t-sql/database-console-commands/dbcc-checkdb-transact-sql?view=sql-server-ver17
- SQL Server index maintenance: https://learn.microsoft.com/en-us/sql/relational-databases/indexes/reorganize-and-rebuild-indexes?view=sql-server-ver17
- SQL Server Always Encrypted: https://learn.microsoft.com/en-us/sql/relational-databases/security/encryption/always-encrypted-database-engine?view=sql-server-ver17
- PostgreSQL monitoring views: https://www.postgresql.org/docs/current/monitoring-stats.html
- PostgreSQL EXPLAIN: https://www.postgresql.org/docs/current/using-explain.html
- PostgreSQL backup and restore: https://www.postgresql.org/docs/current/backup.html
- PostgreSQL SCRAM auth: https://www.postgresql.org/docs/current/auth-password.html
- PostgreSQL replication: https://www.postgresql.org/docs/current/warm-standby.html
- PostgreSQL vacuuming: https://www.postgresql.org/docs/current/routine-vacuuming.html

