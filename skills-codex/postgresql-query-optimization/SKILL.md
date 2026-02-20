---
name: postgresql-query-optimization
description: Optimize PostgreSQL query performance using EXPLAIN analysis, statistics, and index strategy. Use when Codex must investigate slow queries, high I/O SQL, plan regressions, or inefficient join/filter patterns.
---

# postgresql-query-optimization

## Workflow
1. Rank high-impact queries from runtime stats.
2. Capture `EXPLAIN (ANALYZE, BUFFERS)` for representative executions.
3. Identify root causes: bad estimates, join strategy, index mismatch, SQL pattern.
4. Propose low-risk improvements and expected impact.
5. Validate with latency and resource metrics, then document rollback.

## Scope Boundaries
### In Scope
- EXPLAIN-based query diagnosis and optimization strategy.
- Index/statistics/query-pattern improvements.
- Validation using latency and resource metrics.
### Out of Scope
- General database health triage (use postgresql-health-check).
- Backup and PITR architecture (use postgresql-backup-recovery).
- Authentication/privilege hardening (use postgresql-security).

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
- `references/explain-playbook.md`
- `references/index-strategy.md`
- `examples/01-bad-estimates.md`
- `examples/02-pattern-search.md`
- `examples/03-temp-files.md`

## Triggering Note
Use this skill when the request clearly matches this domain and requires platform-specific DBA guidance.
