---
name: sqlserver-backup-recovery
description: Design, audit, and validate SQL Server backup and recovery strategy for explicit RPO/RTO objectives. Use when Codex must review backup coverage, plan restore workflows, or respond to data loss/corruption incidents.
---

# sqlserver-backup-recovery

## Workflow
1. Capture business RPO/RTO and system criticality.
2. Audit full/diff/log backup coverage by recovery model.
3. Verify integrity strategy (checksums, restore verification, restore drills).
4. Define incident-specific restore sequences including PITR.
5. Produce runbook with timings, dependencies, and validation steps.

## Scope Boundaries
### In Scope
- Backup strategy assessment by recovery model and business objectives.
- Restore sequencing, PITR planning, and drill readiness.
- Recoverability risk reporting and runbook creation.
### Out of Scope
- Performance tuning not tied to backup/recovery (use sqlserver-query-optimization).
- AG lag/failover operations (use sqlserver-high-availability).
- Privilege and encryption audits (use sqlserver-security).

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
- `references/backup-strategy-matrix.md`
- `references/restore-runbooks.md`
- `examples/01-pitr.md`
- `examples/02-corruption.md`
- `examples/03-dr-restore.md`

## Triggering Note
Use this skill when the request clearly matches this domain and requires platform-specific DBA guidance.
