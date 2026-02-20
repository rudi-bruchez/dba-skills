---
name: postgresql-health-check
description: Perform PostgreSQL health diagnostics for incidents and proactive operations reviews. Use when Codex must assess locks, workload pressure, WAL/checkpoint behavior, autovacuum health, and overall operational risk.
---

# postgresql-health-check

## Workflow
1. Collect environment and workload baseline.
2. Inspect active sessions, blocking chains, and wait conditions.
3. Assess throughput, cache behavior, and I/O pressure signals.
4. Evaluate autovacuum effectiveness and bloat indicators.
5. Rank findings with severity and confidence.

## Scope Boundaries
### In Scope
- Operational health diagnostics (sessions, locks, waits, resource pressure).
- Autovacuum/WAL/checkpoint risk indicators.
- Incident triage with prioritized next steps.
### Out of Scope
- Detailed SQL rewrite and plan optimization (use postgresql-query-optimization).
- Replication failover operations (use postgresql-replication).
- Auth/privilege hardening projects (use postgresql-security).

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
- `references/diagnostic-sql.md`
- `references/severity-model.md`
- `examples/01-lock-contention.md`
- `examples/02-connection-saturation.md`
- `examples/03-wal-growth.md`

## Triggering Note
Use this skill when the request clearly matches this domain and requires platform-specific DBA guidance.
