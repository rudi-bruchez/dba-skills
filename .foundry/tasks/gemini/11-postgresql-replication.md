# Task: Skill 11 - PostgreSQL Replication

## Objective
Create a specialized agent skill for PostgreSQL Replication that follows the specification at https://agentskills.io/specification.

## Requirements
- **Skill Name:** `postgresql-replication`
- **Description:** Guidance and tools for implementing high availability, read scalability, and disaster recovery through PostgreSQL replication.
- **Tools Included:** Guidance for physical and logical replication, synchronous vs. asynchronous modes, and HA tools (Patroni).
- **Key Concepts:** Streaming replication, logical replication, replication slots, synchronous vs. asynchronous, Patroni, repmgr.

## Steps

### 1. Research & Analysis
- [x] Done in `gemini-research/postgresql-replication.md`.
- Review replication techniques and HA/DR solutions for PostgreSQL 14-17.

### 2. Implementation - Skill Structure
- Create a directory `skills/postgresql-replication/`.
- Create a `SKILL.md` file with YAML frontmatter.
- Define the skill name, description, and metadata.

### 3. Implementation - Content
- Write clear, concise instructions for setting up replication.
- Include essential SQL queries for monitoring replication status and lag.
- Add sections for:
    - Physical vs. Logical Replication
    - Synchronous vs. Asynchronous Replication
    - Replication Slots & WAL Retention
    - High Availability with Patroni or repmgr
    - Monitoring Replication Health & Lag

### 4. Implementation - Scripts & Tools (Optional)
- Add scripts for monitoring replication health in `skills/postgresql-replication/scripts/`.
- Include instructions for using `Patroni`, `repmgr`, and `HAProxy`.

### 5. Implementation - References & Examples
- Add links to PostgreSQL documentation and community resources.
- Provide practical examples of replication failover and read-only routing.

### 6. Validation
- Ensure the skill follows the specification.
- Verify SQL queries for accuracy and correctness.

## Deliverables
- `skills/postgresql-replication/SKILL.md`
- `skills/postgresql-replication/scripts/*.sql` (Optional)
- `skills/postgresql-replication/references/` (Optional)
