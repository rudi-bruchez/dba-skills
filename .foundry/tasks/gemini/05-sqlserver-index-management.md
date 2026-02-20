# Task: Skill 05 - SQL Server Index Management

## Objective
Create a specialized agent skill for SQL Server Index Management that follows the specification at https://agentskills.io/specification.

## Requirements
- **Skill Name:** `sqlserver-index-management`
- **Description:** Guidance and tools for optimizing database performance through efficient index selection and maintenance.
- **Tools Included:** Guidance for index types, fragmentation analysis, and maintenance strategies.
- **Key Concepts:** Clustered vs. non-clustered, Fill Factor, rebuild vs. reorganize, statistics, missing/unused indexes.

## Steps

### 1. Research & Analysis
- [x] Done in `gemini-research/sqlserver-index-management.md`.
- Review modern index types and maintenance techniques for 2024-2025.

### 2. Implementation - Skill Structure
- Create a directory `skills/sqlserver-index-management/`.
- Create a `SKILL.md` file with YAML frontmatter.
- Define the skill name, description, and metadata.

### 3. Implementation - Content
- Write clear, concise instructions for identifying and managing indexes.
- Include essential T-SQL queries for fragmentation analysis and index maintenance.
- Add sections for:
    - Choosing the Right Index Types (Clustered, Non-Clustered, Columnstore)
    - Fragmentation Detection & Maintenance (Reorganize vs. Rebuild)
    - Fill Factor Best Practices
    - Identifying Missing & Unused Indexes
    - Statistics Maintenance

### 4. Implementation - Scripts & Tools (Optional)
- Add scripts for index fragmentation checks and maintenance in `skills/sqlserver-index-management/scripts/`.
- Include instructions for using Ola Hallengren's Index Maintenance solution.

### 5. Implementation - References & Examples
- Add links to Microsoft documentation and community resources.
- Provide practical examples of index creation and maintenance impact.

### 6. Validation
- Ensure the skill follows the specification.
- Verify T-SQL queries for accuracy and performance impact.

## Deliverables
- `skills/sqlserver-index-management/SKILL.md`
- `skills/sqlserver-index-management/scripts/*.sql` (Optional)
- `skills/sqlserver-index-management/references/` (Optional)
