# Task: Skill 09 - PostgreSQL Backup & Recovery

## Objective
Create a specialized agent skill for PostgreSQL Backup & Recovery that follows the specification at https://agentskills.io/specification.

## Requirements
- **Skill Name:** `postgresql-backup-recovery`
- **Description:** Guidance and tools for implementing robust backup and recovery strategies in PostgreSQL.
- **Tools Included:** Backup and restore commands (`pg_dump`, `pg_basebackup`), WAL archiving, and PITR.
- **Key Concepts:** Logical vs. Physical backups, WAL archiving, Point-in-Time Recovery (PITR), checksums, RPO/RTO.

## Steps

### 1. Research & Analysis
- [x] Done in `gemini-research/postgresql-backup-recovery.md`.
- Review modern backup strategies and disaster recovery best practices for PostgreSQL 14-17.

### 2. Implementation - Skill Structure
- Create a directory `skills/postgresql-backup-recovery/`.
- Create a `SKILL.md` file with YAML frontmatter.
- Define the skill name, description, and metadata.

### 3. Implementation - Content
- Write clear, concise instructions for setting up backup and recovery.
- Include essential SQL/Shell commands for backup, restore, and verification.
- Add sections for:
    - Choosing Between Logical & Physical Backups
    - Implementing WAL Archiving & PITR
    - Verifying Backup Integrity & Data Checksums
    - Disaster Recovery & Point-in-Time Restore
    - Best Practices for Backup Storage and Retention

### 4. Implementation - Scripts & Tools (Optional)
- Add scripts for automated backups and restore verification in `skills/postgresql-backup-recovery/scripts/`.
- Include instructions for using `pgBackRest` and `Barman`.

### 5. Implementation - References & Examples
- Add links to PostgreSQL documentation and community resources.
- Provide practical examples of backup and restore scenarios.

### 6. Validation
- Ensure the skill follows the specification.
- Verify SQL/Shell commands for accuracy and correctness.

## Deliverables
- `skills/postgresql-backup-recovery/SKILL.md`
- `skills/postgresql-backup-recovery/scripts/*.sql` (Optional)
- `skills/postgresql-backup-recovery/references/` (Optional)
