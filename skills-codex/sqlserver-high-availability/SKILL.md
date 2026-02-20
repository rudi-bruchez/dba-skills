---
name: sqlserver-high-availability
description: Assess and improve SQL Server high availability and disaster recovery posture, primarily for Always On Availability Groups. Use when Codex must diagnose replica lag, validate failover readiness, or plan HA/DR runbooks.
---

# sqlserver-high-availability

## Workflow
1. Map topology, replica roles, sync modes, and business objectives.
2. Assess health: send queue, redo queue, synchronization state, endpoint status.
3. Validate failover prerequisites, listener behavior, and dependency readiness.
4. Review backup preferences and job placement in AG topology.
5. Produce planned failover and post-failover validation checklist.

## Scope Boundaries
### In Scope
- Always On AG health diagnostics and lag assessment.
- Failover readiness planning and post-failover validation.
- HA/DR risk alignment with RPO/RTO expectations.
### Out of Scope
- Backup chain design details (use sqlserver-backup-recovery).
- Deep query tuning (use sqlserver-query-optimization).
- Security posture hardening (use sqlserver-security).

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
- `references/ag-health-checks.md`
- `references/failover-runbook.md`
- `examples/01-redo-lag.md`
- `examples/02-planned-failover.md`
- `examples/03-dr-risk.md`

## Triggering Note
Use this skill when the request clearly matches this domain and requires platform-specific DBA guidance.
