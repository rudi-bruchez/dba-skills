# Task: Build `postgresql-security` Skill

## Skill Metadata

```yaml
name: postgresql-security
description: Audits and hardens PostgreSQL security including role management, pg_hba.conf configuration, SSL/TLS setup, row-level security, and privilege reviews. Use when performing security audits, onboarding new instances, responding to security incidents, or preparing for compliance reviews.
metadata:
  author: dba-skills
  version: "1.0"
  platform: PostgreSQL 13+
```

## Directory Structure

```
postgresql-security/
├── SKILL.md
├── references/
│   ├── roles-privileges.md      # PostgreSQL role system and privilege model
│   ├── pg-hba-guide.md          # pg_hba.conf configuration and best practices
│   └── rls-guide.md             # Row-level security policies
└── examples/
    └── examples.md              # Security audit and hardening scenarios
```

## SKILL.md Content Plan

### Sections

1. **Security Audit Workflow** - How to assess current security posture
2. **Role & Privilege Management** - Least privilege, role hierarchy
3. **Authentication (pg_hba.conf)** - Connection authentication methods
4. **SSL/TLS Configuration** - Encrypting connections
5. **Row-Level Security** - Implementing data access policies
6. **Audit Logging** - pgaudit extension and log settings
7. **Security Hardening Checklist** - Top misconfigurations to fix

### Key System Views and Functions

- `pg_roles` / `pg_user` - Role/user definitions
- `pg_hba_file_rules` - Current pg_hba.conf rules (PG 10+)
- `information_schema.role_table_grants` - Table privileges
- `information_schema.role_column_grants` - Column privileges
- `pg_namespace` - Schema information
- `pg_class` - Table/view ownership
- `has_table_privilege()` - Check table access
- `has_column_privilege()` - Check column access
- `current_user`, `session_user` - Current identity
- `pg_stat_ssl` - SSL connection status
- `pg_policies` - Row-level security policies
- `pg_stat_activity` - Active connections with user info

### Key Security Areas

**Superuser management:**
- Limit superuser count to minimum (ideally 1)
- Disable superuser remote login
- Use `pg_monitor` role instead for monitoring

**Authentication:**
- Prefer `scram-sha-256` over `md5`
- Avoid `trust` for network connections
- Use SSL certificate authentication for service accounts
- Restrict connection source IPs in pg_hba.conf

**Privilege hardening:**
- Revoke CREATE on public schema (PG 15+ default)
- Use dedicated roles per application/service
- Never use superuser for application connections
- Grant minimum required privileges

**Connection security:**
- Enable SSL: `ssl = on` in postgresql.conf
- Force SSL for sensitive connections in pg_hba.conf
- Set `ssl_min_protocol_version = TLSv1.2`

**Audit logging:**
- Install pgaudit extension
- Set `log_connections = on`
- Set `log_disconnections = on`
- Set `log_duration = on` for sensitive systems
- Configure `log_statement = ddl` minimum

### pg_hba.conf Authentication Methods

| Method | Security | Use Case |
|---|---|---|
| scram-sha-256 | High | Default for most connections |
| md5 | Medium | Legacy compatibility only |
| trust | None | Local unix socket only |
| peer | High | Local connections |
| cert | Very High | Service accounts with certs |
| ldap | Varies | Enterprise directory integration |
| reject | Explicit deny | Block specific hosts/users |

### Progressive Disclosure

- SKILL.md: Audit workflow + top vulnerabilities + links
- `references/roles-privileges.md`: Full privilege model and GRANT syntax
- `references/pg-hba-guide.md`: pg_hba.conf format, methods, examples
- `references/rls-guide.md`: Row-level security policy syntax and patterns
- `examples/examples.md`: Audit findings and remediation scenarios

## References to Research

See `claude-research/postgresql-security.md` for detailed findings.

## Acceptance Criteria

- [ ] Agent can audit current security posture with findings
- [ ] Agent can recommend pg_hba.conf improvements
- [ ] Agent can implement row-level security policies
- [ ] Agent can configure SSL/TLS correctly
- [ ] Examples show common misconfigurations with fixes
