# Task: Skill 07 - PostgreSQL Health Check & Diagnostics

## Objective
Create a specialized agent skill for PostgreSQL Health Check & Diagnostics that follows the specification at https://agentskills.io/specification.

## Requirements
- **Skill Name:** `postgresql-health-check`
- **Description:** Guidance and tools for monitoring PostgreSQL health, identifying performance bottlenecks, and performing regular health checks.
- **Tools Included:** Guidance for using system views (`pg_stat_*`), `pg_stat_statements`, and vacuum monitoring.
- **Key Metrics:** Active connections, transaction rate, cache hit ratio, wait events, table/index bloat, and vacuum status.

## Steps

### 1. Research & Analysis
- [x] Done in `gemini-research/postgresql-health-check.md`.
- Review system views and diagnostic techniques for PostgreSQL 14-17.

### 2. Implementation - Skill Structure
- Create a directory `skills/postgresql-health-check/`.
- Create a `SKILL.md` file with YAML frontmatter.
- Define the skill name, description, and metadata.

### 3. Implementation - Content
- Write clear, concise instructions for each health check metric.
- Include essential SQL queries for monitoring activity and performance.
- Add sections for:
    - Activity & Connection Monitoring
    - Performance & Cache Hit Ratio
    - Wait Events Investigation
    - Table & Index Bloat Analysis
    - Vacuuming & Autovacuum Tuning

### 4. Implementation - Scripts & Tools (Optional)
- Add scripts for common health check queries in `skills/postgresql-health-check/scripts/`.
- Include instructions for using `pgBadger` and `pg_stat_statements`.

### 5. Implementation - References & Examples
- Add links to PostgreSQL documentation and community resources.
- Provide practical examples of interpreting health check results.

### 6. Validation
- Ensure the skill follows the specification.
- Verify SQL queries for accuracy and performance impact.

## Deliverables
- `skills/postgresql-health-check/SKILL.md`
- `skills/postgresql-health-check/scripts/*.sql` (Optional)
- `skills/postgresql-health-check/references/` (Optional)
