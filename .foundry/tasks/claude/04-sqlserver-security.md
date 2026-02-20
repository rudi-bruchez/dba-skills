# Task: Build `sqlserver-security` Skill

## Skill Metadata

```yaml
name: sqlserver-security
description: Audits and hardens SQL Server security including login management, permission reviews, encryption setup, auditing configuration, and compliance checks. Use when performing security reviews, setting up new instances, responding to security incidents, or preparing for compliance audits (SOC 2, PCI-DSS, HIPAA).
metadata:
  author: dba-skills
  version: "1.0"
  platform: SQL Server 2016+
```

## Directory Structure

```
sqlserver-security/
├── SKILL.md
├── references/
│   ├── permissions-model.md     # SQL Server permission hierarchy
│   ├── encryption-guide.md      # TDE, Always Encrypted, column encryption
│   └── audit-setup.md           # SQL Server Audit configuration
└── examples/
    └── examples.md              # Security audit and hardening scenarios
```

## SKILL.md Content Plan

### Sections

1. **Security Audit Workflow** - How to perform a security review
2. **Authentication & Logins** - SA account, mixed mode, orphaned users
3. **Permission Review** - Least privilege, fixed server roles, db_owner overuse
4. **Encryption** - TDE, Always Encrypted, connection encryption
5. **SQL Server Audit** - Setting up audit for compliance
6. **Surface Area Reduction** - Disabling unused features
7. **Vulnerability Checklist** - Common misconfigurations

### Key System Views and Commands

- `sys.server_principals` - Server logins
- `sys.database_principals` - Database users
- `sys.server_role_members` - Server role membership
- `sys.database_role_members` - Database role membership
- `sys.database_permissions` - Explicit permissions
- `sys.server_permissions` - Server-level permissions
- `fn_my_permissions()` - Effective permissions
- `sys.dm_exec_sessions` - Active sessions with login info
- `sys.sql_logins` - SQL logins and password policy
- `sys.credentials` - Credential objects
- `sys.symmetric_keys` - Encryption keys
- `sys.certificates` - Certificates
- `sys.dm_database_encryption_keys` - TDE status
- `sys.server_audits` - Audit objects
- `sp_helprotect` - Permission report

### Security Hardening Items

- Rename/disable SA account
- Disable mixed mode auth (Windows auth only where possible)
- Remove public role permissions from sensitive objects
- Revoke EXECUTE AS from untrusted principals
- Enable TDE for sensitive databases
- Require encrypted connections (FORCE ENCRYPTION)
- Disable: xp_cmdshell, Ole Automation, CLR, Linked Servers if unused
- Enable SQL Server Audit for login failures + DDL changes
- Review sysadmin role members
- Fix orphaned users
- Implement contained databases carefully

### Compliance Focus Areas

- PCI-DSS: encryption of card data, audit logging, access control
- HIPAA: access controls, audit trails, encryption
- SOC 2: access management, change management, availability

### Progressive Disclosure

- SKILL.md: Audit workflow + top vulnerabilities + links to details
- `references/permissions-model.md`: Full permission hierarchy
- `references/encryption-guide.md`: TDE and Always Encrypted setup
- `references/audit-setup.md`: Complete audit configuration guide
- `examples/examples.md`: Security audit findings and remediations

## References to Research

See `claude-research/sqlserver-security.md` for detailed findings.

## Acceptance Criteria

- [ ] Agent can run a complete security audit producing findings list
- [ ] Agent can identify and fix permission issues
- [ ] Agent can set up TDE from scratch
- [ ] Agent can configure SQL Server Audit
- [ ] Examples include realistic misconfigurations and their fixes
