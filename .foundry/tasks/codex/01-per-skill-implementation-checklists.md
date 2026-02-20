# Per-Skill Implementation Checklists

## `sqlserver-health-check`
- Create:
  - `skills-codex/sqlserver-health-check/SKILL.md`
  - `skills-codex/sqlserver-health-check/references/health-check-queries.md`
  - `skills-codex/sqlserver-health-check/references/severity-model.md`
  - `skills-codex/sqlserver-health-check/examples/01-incident-triage.md`
  - `skills-codex/sqlserver-health-check/examples/02-etl-overrun.md`
  - `skills-codex/sqlserver-health-check/examples/03-timeouts.md`
- Validate:
  - Queries are read-only by default.
  - Output includes finding, evidence, impact, next action.

## `sqlserver-query-optimization`
- Create:
  - `skills-codex/sqlserver-query-optimization/SKILL.md`
  - `skills-codex/sqlserver-query-optimization/references/query-store-workflow.md`
  - `skills-codex/sqlserver-query-optimization/references/plan-analysis-playbook.md`
  - `skills-codex/sqlserver-query-optimization/examples/01-regression.md`
  - `skills-codex/sqlserver-query-optimization/examples/02-high-reads.md`
  - `skills-codex/sqlserver-query-optimization/examples/03-memory-spills.md`
- Validate:
  - Every recommendation includes before/after validation metrics.
  - Forced plan guidance includes rollback criteria.

## `sqlserver-backup-recovery`
- Create:
  - `skills-codex/sqlserver-backup-recovery/SKILL.md`
  - `skills-codex/sqlserver-backup-recovery/references/backup-strategy-matrix.md`
  - `skills-codex/sqlserver-backup-recovery/references/restore-runbooks.md`
  - `skills-codex/sqlserver-backup-recovery/examples/01-pitr.md`
  - `skills-codex/sqlserver-backup-recovery/examples/02-corruption.md`
  - `skills-codex/sqlserver-backup-recovery/examples/03-dr-restore.md`
- Validate:
  - Recovery model and log chain logic are explicit.
  - Restore testing is required, not optional.

## `sqlserver-security`
- Create:
  - `skills-codex/sqlserver-security/SKILL.md`
  - `skills-codex/sqlserver-security/references/permissions-audit.md`
  - `skills-codex/sqlserver-security/references/encryption-decision-guide.md`
  - `skills-codex/sqlserver-security/examples/01-overprivileged-login.md`
  - `skills-codex/sqlserver-security/examples/02-sensitive-data.md`
  - `skills-codex/sqlserver-security/examples/03-audit-gaps.md`
- Validate:
  - Remediation sequence is staged.
  - Compatibility caveats are included for encryption recommendations.

## `sqlserver-index-maintenance`
- Create:
  - `skills-codex/sqlserver-index-maintenance/SKILL.md`
  - `skills-codex/sqlserver-index-maintenance/references/index-policy.md`
  - `skills-codex/sqlserver-index-maintenance/references/maintenance-sql.md`
  - `skills-codex/sqlserver-index-maintenance/examples/01-no-op-case.md`
  - `skills-codex/sqlserver-index-maintenance/examples/02-log-risk.md`
  - `skills-codex/sqlserver-index-maintenance/examples/03-partitioned-table.md`
- Validate:
  - Guidance avoids static threshold-only jobs.
  - Log/tempdb risk checks are mandatory before rebuild actions.

## `sqlserver-high-availability`
- Create:
  - `skills-codex/sqlserver-high-availability/SKILL.md`
  - `skills-codex/sqlserver-high-availability/references/ag-health-checks.md`
  - `skills-codex/sqlserver-high-availability/references/failover-runbook.md`
  - `skills-codex/sqlserver-high-availability/examples/01-redo-lag.md`
  - `skills-codex/sqlserver-high-availability/examples/02-planned-failover.md`
  - `skills-codex/sqlserver-high-availability/examples/03-dr-risk.md`
- Validate:
  - Planned vs forced failover path is explicit.
  - Disruptive operations require user confirmation.

## `postgresql-health-check`
- Create:
  - `skills-codex/postgresql-health-check/SKILL.md`
  - `skills-codex/postgresql-health-check/references/diagnostic-sql.md`
  - `skills-codex/postgresql-health-check/references/severity-model.md`
  - `skills-codex/postgresql-health-check/examples/01-lock-contention.md`
  - `skills-codex/postgresql-health-check/examples/02-connection-saturation.md`
  - `skills-codex/postgresql-health-check/examples/03-wal-growth.md`
- Validate:
  - Non-superuser visibility limitations are explained.
  - Findings are severity-ranked with confidence.

## `postgresql-query-optimization`
- Create:
  - `skills-codex/postgresql-query-optimization/SKILL.md`
  - `skills-codex/postgresql-query-optimization/references/explain-playbook.md`
  - `skills-codex/postgresql-query-optimization/references/index-strategy.md`
  - `skills-codex/postgresql-query-optimization/examples/01-bad-estimates.md`
  - `skills-codex/postgresql-query-optimization/examples/02-pattern-search.md`
  - `skills-codex/postgresql-query-optimization/examples/03-temp-files.md`
- Validate:
  - EXPLAIN-based reasoning is required.
  - Recommendations include write overhead tradeoffs.

## `postgresql-backup-recovery`
- Create:
  - `skills-codex/postgresql-backup-recovery/SKILL.md`
  - `skills-codex/postgresql-backup-recovery/references/backup-methods.md`
  - `skills-codex/postgresql-backup-recovery/references/recovery-runbooks.md`
  - `skills-codex/postgresql-backup-recovery/examples/01-pitr.md`
  - `skills-codex/postgresql-backup-recovery/examples/02-cluster-restore.md`
  - `skills-codex/postgresql-backup-recovery/examples/03-logical-recovery.md`
- Validate:
  - WAL archive dependency is explicit.
  - Recoverability claims require tested drills.

## `postgresql-security`
- Create:
  - `skills-codex/postgresql-security/SKILL.md`
  - `skills-codex/postgresql-security/references/auth-hardening.md`
  - `skills-codex/postgresql-security/references/role-governance.md`
  - `skills-codex/postgresql-security/examples/01-scram-migration.md`
  - `skills-codex/postgresql-security/examples/02-default-privileges.md`
  - `skills-codex/postgresql-security/examples/03-public-schema.md`
- Validate:
  - MD5 to SCRAM migration path includes rollback.
  - Superuser minimization strategy is concrete.

## `postgresql-replication`
- Create:
  - `skills-codex/postgresql-replication/SKILL.md`
  - `skills-codex/postgresql-replication/references/replication-diagnostics.md`
  - `skills-codex/postgresql-replication/references/failover-runbook.md`
  - `skills-codex/postgresql-replication/examples/01-slot-wal-growth.md`
  - `skills-codex/postgresql-replication/examples/02-lag-burst.md`
  - `skills-codex/postgresql-replication/examples/03-controlled-switchover.md`
- Validate:
  - Slot risk monitoring is mandatory.
  - Post-failover backup and routing checks are included.

