# Codex Skill Research Matrix

## `sqlserver-health-check`
### Purpose
Deliver incident-ready SQL Server health diagnostics with ranked findings.
### Knowledge Required
- Version and feature context
- Waits, blocking, memory and I/O DMVs
- Backup freshness and integrity evidence
- CHECKDB cadence and status
### Research Notes
- Use Query Store + DMVs for current and historical signals.
- Avoid single-snapshot conclusions when counters are volatile.
- Always provide confidence and unknowns when permissions are limited.
### Required References in Final Skill
- Read-only query pack
- Severity scoring model
- Report template (exec summary + findings table)
### Example Scenarios
- Morning performance degradation
- ETL window overrun
- Intermittent timeout spikes

## `sqlserver-query-optimization`
### Purpose
Resolve SQL Server query latency/regression using plans + Query Store.
### Knowledge Required
- Query Store collection and regression analysis
- Plan shape interpretation and cardinality issues
- Parameter sniffing and plan instability patterns
- Index/statistics/query rewrite tradeoffs
### Research Notes
- Use forced plans tactically with exit criteria.
- Prioritize SARGability, stats quality, and indexing before hints.
- Require before/after metrics (duration, CPU, reads, spills).
### Required References in Final Skill
- Plan anti-pattern catalog
- Query Store workflow playbook
- Validation and rollback checklist
### Example Scenarios
- Post-release regression
- Heavy logical reads from non-sargable predicates
- Hash/sort spill-driven latency

## `sqlserver-backup-recovery`
### Purpose
Design and validate recovery strategy for explicit RPO/RTO.
### Knowledge Required
- Recovery models and log chain behavior
- Full/diff/log orchestration
- Verification vs recoverability limits
- Restore sequence and PITR workflow
### Research Notes
- Backups need checksums and tested restore drills.
- Encryption, retention, and offsite copies are baseline.
- Runbook quality is as important as backup success state.
### Required References in Final Skill
- Strategy matrix by recovery model
- Restore runbooks by incident type
- Drill evidence template
### Example Scenarios
- Point-in-time restore after accidental delete
- Corruption response path
- Regional outage recovery

## `sqlserver-security`
### Purpose
Audit and harden SQL Server permissions and data protection controls.
### Knowledge Required
- Login/user/role mapping
- Privilege inheritance and elevated role usage
- Always Encrypted, TDE, TLS considerations
- Audit trail and detection coverage
### Research Notes
- Enforce least privilege and reduce shared high-privilege identities.
- Document compatibility impacts for encryption controls.
- Stage permission cleanup to avoid production breakage.
### Required References in Final Skill
- Permission audit SQL
- Encryption decision guide
- Remediation wave plan
### Example Scenarios
- Overprivileged service identity
- Sensitive column protection retrofit
- Audit coverage gap analysis

## `sqlserver-index-maintenance`
### Purpose
Run workload-aware index maintenance with controlled operational risk.
### Knowledge Required
- Fragmentation and page density interpretation
- Rebuild vs reorganize constraints
- Fill factor and write amplification
- Maintenance impact on log/tempdb/HA
### Research Notes
- Avoid static threshold-only maintenance jobs.
- Tie maintenance decisions to user-visible workload impact.
- Pair with stats maintenance strategy.
### Required References in Final Skill
- Decision matrix (do nothing/reorg/rebuild)
- Adaptive SQL scripts
- Risk gate checklist
### Example Scenarios
- High fragmentation but no query pain
- Log growth risk from rebuild
- Large partitioned table strategy

## `sqlserver-high-availability`
### Purpose
Assess and improve Always On AG operational readiness.
### Knowledge Required
- Replica states and sync modes
- Lag indicators (send/redo queues)
- Failover prerequisites and verification
- Backup preference behavior in AG
### Research Notes
- Separate diagnostics from disruptive failover actions.
- Include application routing and timeout validation.
- Test runbooks regularly, not only during incidents.
### Required References in Final Skill
- AG health query set
- Planned failover checklist
- Post-failover validation checklist
### Example Scenarios
- Redo lag growth
- Planned failover rehearsal
- DR async lag risk report

## `postgresql-health-check`
### Purpose
Perform structured PostgreSQL health diagnostics with severity-ranked output.
### Knowledge Required
- `pg_stat_activity`, `pg_stat_database`, lock/wait visibility
- WAL/checkpoint pressure indicators
- Autovacuum and bloat early warning signals
- Backup/replication status basics
### Research Notes
- Keep default diagnostics read-only.
- Trend-based interpretation is preferred over point snapshots.
- Include visibility caveats when privilege scope is limited.
### Required References in Final Skill
- Diagnostic SQL pack
- Severity rubric
- Reporting format
### Example Scenarios
- Lock pile-up
- Connection saturation
- WAL growth anomaly

## `postgresql-query-optimization`
### Purpose
Tune PostgreSQL query performance through EXPLAIN-driven analysis.
### Knowledge Required
- `EXPLAIN (ANALYZE, BUFFERS)`
- Join strategy and estimate mismatch diagnosis
- Index type selection (btree/gin/gist/brin)
- `pg_stat_statements` workload ranking
### Research Notes
- Prefer query/schema fixes before broad parameter changes.
- Validate recommendations with p95/p99 and resource metrics.
- Avoid global planner-disable shortcuts.
### Required References in Final Skill
- EXPLAIN interpretation playbook
- Index strategy guide
- Validation checklist
### Example Scenarios
- Bad estimates driving nested loop explosion
- Pattern search index redesign
- Temp-file-heavy sort reduction

## `postgresql-backup-recovery`
### Purpose
Provide robust backup and recoverability guidance for PostgreSQL.
### Knowledge Required
- Logical vs physical backup boundaries
- WAL archiving requirements for PITR
- Recovery target controls
- Drill execution and restore verification
### Research Notes
- Backup success without restore testing is insufficient.
- Combine methods where business objectives require it.
- Capture measured restore times to validate RTO claims.
### Required References in Final Skill
- Method selection matrix
- Recovery runbooks
- Drill result template
### Example Scenarios
- PITR after bad deployment
- Cluster restore after storage failure
- Single-database logical recovery

## `postgresql-security`
### Purpose
Harden PostgreSQL authN/authZ and reduce exposure.
### Knowledge Required
- `pg_hba.conf` evaluation order
- SCRAM-SHA-256 migration approach
- Role/default privilege governance
- TLS and logging controls
### Research Notes
- Migrate away from MD5 where feasible.
- Minimize superuser and public-schema attack surfaces.
- Use staged rollout to avoid auth lockouts.
### Required References in Final Skill
- Auth hardening checklist
- Role governance SQL
- Change rollback procedures
### Example Scenarios
- MD5 to SCRAM migration
- Default privilege cleanup
- Public schema/function access reduction

## `postgresql-replication`
### Purpose
Operate and troubleshoot PostgreSQL streaming replication and failover readiness.
### Knowledge Required
- Sender/receiver/replay lag model
- Replication slots and WAL retention risk
- Synchronous vs asynchronous durability tradeoffs
- Promotion/failover procedure and validation
### Research Notes
- Slot monitoring is critical to disk safety.
- Separate lag diagnostics from failover execution.
- Post-failover checks must include backup continuity.
### Required References in Final Skill
- Replication diagnostic SQL toolkit
- Switchover/failover runbook
- SLO/alert threshold guidance
### Example Scenarios
- Slot-induced WAL disk growth
- Replica lag during burst load
- Controlled switchover rehearsal

