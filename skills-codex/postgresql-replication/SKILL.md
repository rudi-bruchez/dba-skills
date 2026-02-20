---
name: postgresql-replication
description: Diagnose and operate PostgreSQL replication and failover readiness. Use when Codex must investigate lag, slot-related WAL growth, standby health, switchover planning, or promotion runbooks.
---

# postgresql-replication

## Workflow
1. Map replication topology and durability objectives.
2. Measure sender, receiver, and replay lag.
3. Audit replication slots and WAL retention risk.
4. Validate switchover/failover procedures and prerequisites.
5. Define post-transition checks for routing, backups, and data continuity.

## Scope Boundaries
### In Scope
- Streaming replication lag analysis and slot/WAL risk management.
- Switchover/failover readiness and runbook validation.
- Post-transition verification for routing and continuity.
### Out of Scope
- Backup policy design not specific to replication (use postgresql-backup-recovery).
- SQL plan tuning (use postgresql-query-optimization).
- Authentication and role hardening (use postgresql-security).

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
- `references/replication-diagnostics.md`
- `references/failover-runbook.md`
- `examples/01-slot-wal-growth.md`
- `examples/02-lag-burst.md`
- `examples/03-controlled-switchover.md`

## Triggering Note
Use this skill when the request clearly matches this domain and requires platform-specific DBA guidance.
