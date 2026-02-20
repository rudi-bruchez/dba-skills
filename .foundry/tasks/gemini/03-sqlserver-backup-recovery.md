# Task: Skill 03 - SQL Server Backup & Recovery

## Objective
Create a specialized agent skill for SQL Server Backup & Recovery that follows the specification at https://agentskills.io/specification.

## Requirements
- **Skill Name:** `sqlserver-backup-recovery`
- **Description:** Guidance and tools for implementing robust backup and recovery strategies in SQL Server.
- **Tools Included:** Backup and restore commands, recovery models, and maintenance plans.
- **Key Concepts:** Recovery models, backup types, verification, disaster recovery, and RPO/RTO.

## Steps

### 1. Research & Analysis
- [x] Done in `gemini-research/sqlserver-backup-recovery.md`.
- Review modern backup strategies and disaster recovery best practices for 2024-2025.

### 2. Implementation - Skill Structure
- Create a directory `skills/sqlserver-backup-recovery/`.
- Create a `SKILL.md` file with YAML frontmatter.
- Define the skill name, description, and metadata.

### 3. Implementation - Content
- Write clear, concise instructions for setting up backup and recovery.
- Include essential T-SQL commands for backup, restore, and verification.
- Add sections for:
    - Choosing a Recovery Model
    - Implementing a Backup Strategy (Full, Diff, Log)
    - Verifying Backup Integrity
    - Disaster Recovery & Point-in-Time Restore
    - Best Practices for Backup Storage and Retention

### 4. Implementation - Scripts & Tools (Optional)
- Add scripts for automated backups and restore verification in `skills/sqlserver-backup-recovery/scripts/`.
- Include instructions for using Ola Hallengren's Maintenance Solution.

### 5. Implementation - References & Examples
- Add links to Microsoft documentation and community resources.
- Provide practical examples of backup and restore scenarios.

### 6. Validation
- Ensure the skill follows the specification.
- Verify T-SQL commands for accuracy and correctness.

## Deliverables
- `skills/sqlserver-backup-recovery/SKILL.md`
- `skills/sqlserver-backup-recovery/scripts/*.sql` (Optional)
- `skills/sqlserver-backup-recovery/references/` (Optional)
