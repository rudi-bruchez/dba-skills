# Task: Skill 06 - SQL Server High Availability

## Objective
Create a specialized agent skill for SQL Server High Availability that follows the specification at https://agentskills.io/specification.

## Requirements
- **Skill Name:** `sqlserver-high-availability`
- **Description:** Guidance and tools for implementing high availability and disaster recovery solutions in SQL Server.
- **Tools Included:** Guidance for Always On Availability Groups, Failover Cluster Instances, and Log Shipping.
- **Key Concepts:** Always On AGs, FCIs, Log Shipping, Quorum/Witness, RPO/RTO.

## Steps

### 1. Research & Analysis
- [x] Done in `gemini-research/sqlserver-high-availability.md`.
- Review HA/DR solutions and modern configurations for 2024-2025.

### 2. Implementation - Skill Structure
- Create a directory `skills/sqlserver-high-availability/`.
- Create a `SKILL.md` file with YAML frontmatter.
- Define the skill name, description, and metadata.

### 3. Implementation - Content
- Write clear, concise instructions for setting up HA/DR.
- Include essential T-SQL queries for monitoring AGs, FCIs, and Log Shipping.
- Add sections for:
    - Always On Availability Groups (Config & Best Practices)
    - Failover Cluster Instances (Setup & Maintenance)
    - Log Shipping for Disaster Recovery
    - Quorum & Witness Configuration
    - Read-Only Routing & Multi-Subnet Failover

### 4. Implementation - Scripts & Tools (Optional)
- Add scripts for monitoring AG health and status in `skills/sqlserver-high-availability/scripts/`.
- Include instructions for using Failover Cluster Manager and SSMS tools.

### 5. Implementation - References & Examples
- Add links to Microsoft documentation and community resources.
- Provide practical examples of HA failover and DR recovery scenarios.

### 6. Validation
- Ensure the skill follows the specification.
- Verify T-SQL queries for accuracy and correctness.

## Deliverables
- `skills/sqlserver-high-availability/SKILL.md`
- `skills/sqlserver-high-availability/scripts/*.sql` (Optional)
- `skills/sqlserver-high-availability/references/` (Optional)
