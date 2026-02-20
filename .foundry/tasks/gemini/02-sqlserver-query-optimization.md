# Task: Skill 02 - SQL Server Query Optimization

## Objective
Create a specialized agent skill for SQL Server Query Optimization that follows the specification at https://agentskills.io/specification.

## Requirements
- **Skill Name:** `sqlserver-query-optimization`
- **Description:** Comprehensive guidance and tools for identifying and improving the performance of T-SQL queries.
- **Tools Included:** Guidance for analyzing execution plans, statistics, and Query Store.
- **Key Concepts:** Execution plans, statistics, indexing, Query Store, parameter sniffing, SARGability, and implicit conversions.

## Steps

### 1. Research & Analysis
- [x] Done in `gemini-research/sqlserver-query-optimization.md`.
- Review query tuning techniques and Intelligent Query Processing (IQP) features in SQL Server 2024-2025.

### 2. Implementation - Skill Structure
- Create a directory `skills/sqlserver-query-optimization/`.
- Create a `SKILL.md` file with YAML frontmatter.
- Define the skill name, description, and metadata.

### 3. Implementation - Content
- Write clear, concise instructions for identifying slow queries.
- Include essential T-SQL queries for analyzing execution stats and plans.
- Add sections for:
    - Execution Plan Analysis
    - Statistics Maintenance
    - Indexing Strategies for Query Performance
    - Query Store Best Practices
    - Identifying & Fixing Common Performance Anti-Patterns

### 4. Implementation - Scripts & Tools (Optional)
- Add scripts for identifying expensive queries in `skills/sqlserver-query-optimization/scripts/`.
- Include instructions for using Query Store and graphical plan analysis tools.

### 5. Implementation - References & Examples
- Add links to Microsoft documentation and community resources.
- Provide practical examples of query rewriting and optimization results.

### 6. Validation
- Ensure the skill follows the specification.
- Verify T-SQL queries for accuracy and performance impact.

## Deliverables
- `skills/sqlserver-query-optimization/SKILL.md`
- `skills/sqlserver-query-optimization/scripts/*.sql` (Optional)
- `skills/sqlserver-query-optimization/references/` (Optional)
