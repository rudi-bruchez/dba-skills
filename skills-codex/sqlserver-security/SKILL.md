---
name: sqlserver-security
description: Audit and harden SQL Server security controls including permissions, encryption, and auditing. Use when Codex must perform privilege review, sensitive data protection planning, or phased hardening remediation.
---

# sqlserver-security

## Workflow
1. Inventory principals, role memberships, and privileged paths.
2. Identify over-privileged accounts and broad grants.
3. Evaluate encryption posture for data at rest/in transit/in use.
4. Assess audit coverage and incident traceability.
5. Deliver staged remediation plan with compatibility notes.

## Scope Boundaries
### In Scope
- Authentication/authorization review and privilege minimization.
- Encryption control planning (TLS, TDE, Always Encrypted).
- Audit coverage and staged security remediation.
### Out of Scope
- General health incident triage (use sqlserver-health-check).
- Query latency optimization work (use sqlserver-query-optimization).
- Backup architecture design (use sqlserver-backup-recovery).

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
- `references/permissions-audit.md`
- `references/encryption-decision-guide.md`
- `examples/01-overprivileged-login.md`
- `examples/02-sensitive-data.md`
- `examples/03-audit-gaps.md`

## Triggering Note
Use this skill when the request clearly matches this domain and requires platform-specific DBA guidance.
