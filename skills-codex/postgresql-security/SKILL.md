---
name: postgresql-security
description: Audit and improve PostgreSQL security posture across authentication, roles/privileges, and exposure controls. Use when Codex must assess `pg_hba.conf`, migrate auth methods, tighten privileges, or plan security hardening.
---

# postgresql-security

## Workflow
1. Review authentication paths and `pg_hba.conf` rule order.
2. Audit roles, memberships, and default privileges.
3. Evaluate transport security and logging/audit coverage.
4. Plan phased remediation for auth and privilege changes.
5. Include rollback strategy for potentially disruptive changes.

## Scope Boundaries
### In Scope
- Authentication hardening (`pg_hba.conf`, SCRAM migration).
- Role/default privilege audit and least-privilege remediation.
- Security control and rollback planning for safe rollout.
### Out of Scope
- Replication diagnostics and failover planning (use postgresql-replication).
- Slow-query tuning tasks (use postgresql-query-optimization).
- Backup/recovery design (use postgresql-backup-recovery).

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
- `references/auth-hardening.md`
- `references/role-governance.md`
- `examples/01-scram-migration.md`
- `examples/02-default-privileges.md`
- `examples/03-public-schema.md`

## Triggering Note
Use this skill when the request clearly matches this domain and requires platform-specific DBA guidance.
