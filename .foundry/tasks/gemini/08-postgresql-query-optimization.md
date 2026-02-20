# Task: Skill 08 - PostgreSQL Query Optimization

## Objective
Create a specialized agent skill for PostgreSQL Query Optimization that follows the specification at https://agentskills.io/specification.

## Requirements
- **Skill Name:** `postgresql-query-optimization`
- **Description:** Guidance and tools for analyzing and improving the performance of PostgreSQL SQL queries.
- **Tools Included:** Guidance for analyzing execution plans (`EXPLAIN`), indexing strategies, and server configuration.
- **Key Concepts:** `EXPLAIN (ANALYZE, BUFFERS)`, B-tree/GIN/GiST/BRIN indexes, query rewriting, configuration tuning (`shared_buffers`, `work_mem`).

## Steps

### 1. Research & Analysis
- [x] Done in `gemini-research/postgresql-query-optimization.md`.
- Review query planning, indexing, and performance tuning for PostgreSQL 14-17.

### 2. Implementation - Skill Structure
- Create a directory `skills/postgresql-query-optimization/`.
- Create a `SKILL.md` file with YAML frontmatter.
- Define the skill name, description, and metadata.

### 3. Implementation - Content
- Write clear, concise instructions for identifying slow queries.
- Include essential SQL queries for analyzing execution plans and stats.
- Add sections for:
    - Execution Plan Analysis (`EXPLAIN ANALYZE`)
    - Choosing the Right Index Types (B-tree, GIN, GiST, BRIN)
    - Query Rewriting & Anti-Patterns
    - Server Configuration for Query Performance
    - Identifying & Fixing Common Performance Bottlenecks

### 4. Implementation - Scripts & Tools (Optional)
- Add scripts for identifying expensive queries in `skills/postgresql-query-optimization/scripts/`.
- Include instructions for using `pg_stat_statements` and visualizers for JSON plans.

### 5. Implementation - References & Examples
- Add links to PostgreSQL documentation and community resources.
- Provide practical examples of query rewriting and optimization results.

### 6. Validation
- Ensure the skill follows the specification.
- Verify SQL queries for accuracy and performance impact.

## Deliverables
- `skills/postgresql-query-optimization/SKILL.md`
- `skills/postgresql-query-optimization/scripts/*.sql` (Optional)
- `skills/postgresql-query-optimization/references/` (Optional)
