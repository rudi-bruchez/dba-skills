---
name: sqlserver-query-optimization
description: Optimize SQL Server query performance and resolve regressions using Query Store, execution plans, and indexing/statistics analysis. Use when Codex must diagnose slow SQL, high CPU queries, excessive reads, or plan instability.
---

# sqlserver-query-optimization

## Workflow
1. Capture high-impact query candidates from Query Store and runtime evidence.
2. Compare before/after behavior for regressions.
3. Inspect actual plan operators, cardinality errors, and memory grants.
4. Propose minimal-risk fixes in this order: query rewrite, index/statistics, targeted hints.
5. Validate with objective metrics and provide rollback instructions.

## Scope Boundaries
### In Scope
- Slow query diagnosis with Query Store and execution plans.
- Regression analysis and targeted query/index/statistics tuning.
- Before/after performance validation with rollback path.
### Out of Scope
- Broad instance health triage without query focus (use sqlserver-health-check).
- Backup/recovery policy design (use sqlserver-backup-recovery).
- Permission or encryption hardening (use sqlserver-security).

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
- `references/query-store-workflow.md`
- `references/plan-analysis-playbook.md`
- `examples/01-regression.md`
- `examples/02-high-reads.md`
- `examples/03-memory-spills.md`

## Triggering Note
Use this skill when the request clearly matches this domain and requires platform-specific DBA guidance.
