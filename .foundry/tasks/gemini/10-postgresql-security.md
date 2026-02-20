# Task: Skill 10 - PostgreSQL Security

## Objective
Create a specialized agent skill for PostgreSQL Security that follows the specification at https://agentskills.io/specification.

## Requirements
- **Skill Name:** `postgresql-security`
- **Description:** Guidance and tools for protecting PostgreSQL data through authentication, authorization, and encryption.
- **Tools Included:** Guidance for `pg_hba.conf`, roles, users, and security extensions (`pgaudit`).
- **Key Concepts:** Authentication (`scram-sha-256`, `md5`), authorization (RBAC), Row-Level Security (RLS), SSL/TLS, and auditing.

## Steps

### 1. Research & Analysis
- [x] Done in `gemini-research/postgresql-security.md`.
- Review modern security practices and compliance requirements for PostgreSQL 14-17.

### 2. Implementation - Skill Structure
- Create a directory `skills/postgresql-security/`.
- Create a `SKILL.md` file with YAML frontmatter.
- Define the skill name, description, and metadata.

### 3. Implementation - Content
- Write clear, concise instructions for securing PostgreSQL.
- Include essential SQL queries for managing roles, users, and permissions.
- Add sections for:
    - Authentication Best Practices (`pg_hba.conf`, SCRAM-SHA-256)
    - Implementing Least Privilege (RBAC)
    - Data Encryption (SSL/TLS, pgcrypto, File-System Level)
    - Row-Level Security (RLS) Best Practices
    - Auditing & Monitoring (pgaudit, logging settings)

### 4. Implementation - Scripts & Tools (Optional)
- Add scripts for security checks and audits in `skills/postgresql-security/scripts/`.
- Include instructions for using `pgaudit` and security checklists.

### 5. Implementation - References & Examples
- Add links to PostgreSQL documentation and industry standards.
- Provide practical examples of securing a database.

### 6. Validation
- Ensure the skill follows the specification.
- Verify SQL queries for accuracy and security effectiveness.

## Deliverables
- `skills/postgresql-security/SKILL.md`
- `skills/postgresql-security/scripts/*.sql` (Optional)
- `skills/postgresql-security/references/` (Optional)
