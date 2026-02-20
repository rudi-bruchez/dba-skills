---
name: sqlserver-index-maintenance
description: Plan and tune SQL Server index maintenance using workload-aware criteria instead of blanket fragmentation jobs. Use when Codex must evaluate rebuild/reorganize actions, maintenance windows, and log/tempdb risk.
---

# sqlserver-index-maintenance

## Workflow
1. Collect workload evidence and identify impacted indexes.
2. Evaluate fragmentation, page density, and query impact.
3. Decide among no-op, reorganize, rebuild, or redesign.
4. Estimate operational cost (log, tempdb, blocking, HA impact).
5. Validate post-maintenance effect and adjust policy.

## Scope Boundaries
### In Scope
- Index maintenance policy review with workload-aware decisions.
- Rebuild/reorganize/no-op decisioning and risk gating.
- Operational impact analysis on log, tempdb, and maintenance windows.
### Out of Scope
- Operator-level query rewrites (use sqlserver-query-optimization).
- Availability group failover work (use sqlserver-high-availability).
- Security remediation (use sqlserver-security).

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
- `references/index-policy.md`
- `references/maintenance-sql.md`
- `examples/01-no-op-case.md`
- `examples/02-log-risk.md`
- `examples/03-partitioned-table.md`

## Triggering Note
Use this skill when the request clearly matches this domain and requires platform-specific DBA guidance.
