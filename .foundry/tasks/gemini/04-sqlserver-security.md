# Task: Skill 04 - SQL Server Security

## Objective
Create a specialized agent skill for SQL Server Security that follows the specification at https://agentskills.io/specification.

## Requirements
- **Skill Name:** `sqlserver-security`
- **Description:** Guidance and tools for protecting SQL Server data through authentication, authorization, and encryption.
- **Tools Included:** Guidance for logins, users, roles, and encryption features (TDE, Always Encrypted).
- **Key Concepts:** Authentication, authorization, least privilege, encryption (at rest/transit), and auditing.

## Steps

### 1. Research & Analysis
- [x] Done in `gemini-research/sqlserver-security.md`.
- Review modern security practices and compliance requirements for 2024-2025.

### 2. Implementation - Skill Structure
- Create a directory `skills/sqlserver-security/`.
- Create a `SKILL.md` file with YAML frontmatter.
- Define the skill name, description, and metadata.

### 3. Implementation - Content
- Write clear, concise instructions for securing SQL Server.
- Include essential T-SQL queries for managing logins, users, and permissions.
- Add sections for:
    - Authentication Best Practices (Windows vs. SQL vs. Entra ID)
    - Implementing Least Privilege (RBAC)
    - Data Encryption (TDE, SSL/TLS, Always Encrypted)
    - Auditing & Monitoring (SQL Audit, Extended Events)
    - SQL Injection Prevention & Surface Area Reduction

### 4. Implementation - Scripts & Tools (Optional)
- Add scripts for security checks and audits in `skills/sqlserver-security/scripts/`.
- Include instructions for using community security auditing tools.

### 5. Implementation - References & Examples
- Add links to Microsoft documentation and industry standards (e.g., CIS benchmarks).
- Provide practical examples of securing a database.

### 6. Validation
- Ensure the skill follows the specification.
- Verify T-SQL queries for accuracy and security effectiveness.

## Deliverables
- `skills/sqlserver-security/SKILL.md`
- `skills/sqlserver-security/scripts/*.sql` (Optional)
- `skills/sqlserver-security/references/` (Optional)
