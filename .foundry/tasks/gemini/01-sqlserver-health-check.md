# Task: Skill 01 - SQL Server Health Check & Diagnostics

## Objective
Create a specialized agent skill for SQL Server Health Checks & Diagnostics that follows the specification at https://agentskills.io/specification.

## Requirements
- **Skill Name:** `sqlserver-health-check`
- **Description:** Provides comprehensive guidance and tools for monitoring SQL Server health, diagnosing performance issues, and performing regular health checks.
- **Tools Included:** Guidance for using DMVs, `sp_WhoIsActive`, `sp_Blitz`, and Query Store.
- **Key Metrics:** CPU usage, memory health (PLE, Buffer Cache Hit Ratio), I/O latency, wait statistics, and blocking.

## Steps

### 1. Research & Analysis
- [x] Done in `gemini-research/sqlserver-health-check.md`.
- Review existing DMVs and diagnostic queries for SQL Server 2024-2025.

### 2. Implementation - Skill Structure
- Create a directory `skills/sqlserver-health-check/`.
- Create a `SKILL.md` file with YAML frontmatter.
- Define the skill name, description, and metadata.

### 3. Implementation - Content
- Write clear, concise instructions for each health check metric.
- Include essential DMVs and diagnostic queries (T-SQL).
- Add sections for:
    - CPU & Memory Monitoring
    - I/O Performance Analysis
    - Wait Stats Investigation
    - Blocking & Deadlocks
    - Database & Configuration Checks

### 4. Implementation - Scripts & Tools (Optional)
- Add scripts for common health check queries in `skills/sqlserver-health-check/scripts/`.
- Include instructions for using community tools like `sp_WhoIsActive` and `sp_Blitz`.

### 5. Implementation - References & Examples
- Add links to Microsoft documentation and community resources.
- Provide practical examples of interpreting health check results.

### 6. Validation
- Ensure the skill follows the specification.
- Verify T-SQL queries for accuracy and performance impact.

## Deliverables
- `skills/sqlserver-health-check/SKILL.md`
- `skills/sqlserver-health-check/scripts/*.sql` (Optional)
- `skills/sqlserver-health-check/references/` (Optional)
