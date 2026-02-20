---
name: sqlserver-health-check
description: Perform SQL Server health diagnostics for performance incidents, instability, and proactive operational reviews. Use when Codex must assess instance/database health, collect evidence from DMVs, prioritize risks, and recommend safe remediation steps.
---

# sqlserver-health-check

## Workflow
1. Collect context: version, edition, uptime, workload profile, recent changes.
2. Run read-only diagnostics for waits, blocking, CPU, memory, I/O, and database states.
3. Evaluate backup freshness, recovery model coverage, and integrity check cadence.
4. Rank findings by impact and urgency with confidence levels.
5. Recommend immediate actions and medium-term hardening tasks.

## Scope Boundaries
### In Scope
- Instance/database health diagnostics (waits, blocking, CPU, memory, I/O).
- Backup freshness and integrity-check posture review.
- Severity-ranked triage and remediation planning.
### Out of Scope
- Deep query rewrites and operator-level tuning (use sqlserver-query-optimization).
- HA failover execution planning (use sqlserver-high-availability).
- Security hardening projects (use sqlserver-security).

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
- `references/health-check-queries.md`
- `references/severity-model.md`
- `examples/01-incident-triage.md`
- `examples/02-etl-overrun.md`
- `examples/03-timeouts.md`

## Triggering Note
Use this skill when the request clearly matches this domain and requires platform-specific DBA guidance.
