---
name: postgresql-backup-recovery
description: Plan and validate PostgreSQL backup/recovery strategy for logical and physical restore scenarios. Use when Codex must review backup reliability, design PITR workflows, or prepare incident recovery runbooks.
---

# postgresql-backup-recovery

## Workflow
1. Capture RPO/RTO objectives and critical systems.
2. Select backup method: logical, physical, or hybrid.
3. Validate WAL archiving and retention requirements for PITR.
4. Define incident runbooks and recovery targets.
5. Require periodic restore drills with recorded outcomes.

## Scope Boundaries
### In Scope
- Logical/physical backup strategy selection and validation.
- PITR readiness, WAL archival dependencies, and recovery runbooks.
- Restore drill planning and recoverability evidence.
### Out of Scope
- Replication lag root cause and failover operations (use postgresql-replication).
- Query optimization tasks (use postgresql-query-optimization).
- Role/auth hardening (use postgresql-security).

## Safety Rules
- Default to read-only diagnostics before any change.
- Require explicit user confirmation for disruptive or irreversible actions.
- State assumptions when visibility is incomplete.
- Include rollback or fallback guidance for medium/high-risk actions.

## Required Output Shape
- Summary of top findings or objectives.
- Evidence used (queries/metrics/events).
- Recommended actions ranked by impact and risk.
- Validation plan and residual risks.

## Resources
- `references/backup-methods.md`
- `references/recovery-runbooks.md`
- `examples/01-pitr.md`
- `examples/02-cluster-restore.md`
- `examples/03-logical-recovery.md`

## Triggering Note
Use this skill when the request clearly matches this domain and requires platform-specific DBA guidance.
