# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Nature

This is a **knowledge base repository**, not a traditional software project. There is no build process, no tests to run, and no dependencies to install. All content is Markdown with SQL code examples.

## Purpose

A structured collection of AI agent skills for Database Administration (DBA) tasks, covering **SQL Server** (6 skills) and **PostgreSQL** (5 skills). Skills are designed to be consumed by AI agents performing tasks like health checks, query optimization, backup/recovery, security audits, and index management.

## Repository Structure

```
skills/          # Completed agent skills (10 total)
research/claude/ # Research documents used to build skills
tasks/claude/    # Implementation task plans (start with 00-overview.md)
```

Each skill directory follows this layout:
```
skills/<skill-name>/
├── SKILL.md          # Primary instructional file (<500 lines, YAML frontmatter)
├── references/       # Deep-dive technical guides and diagnostic SQL queries
└── examples/         # Concrete scenarios and example outputs
```

## Skill Specification

All skills must conform to the [Agent Skills Specification](https://agentskills.io/specification):

**SKILL.md frontmatter requirements:**
- `name`: max 64 chars, lowercase + hyphens only
- `description`: max 1024 chars, third person, explains what + when to use
- Body: under 500 lines, use progressive disclosure, link details to `references/`

**Content standards:**
- All SQL must be tested and valid
- Use specific DMV/system view names (e.g., `sys.dm_exec_requests`, `pg_stat_activity`)
- Include numeric thresholds and benchmarks
- No abstract descriptions — provide concrete examples
- No date references; use "current best practices" language
- Forward slashes in all file paths

## Implemented Skills

| Platform | Skill |
|---|---|
| SQL Server | `sqlserver-health-check`, `sqlserver-query-optimization`, `sqlserver-backup-recovery`, `sqlserver-security`, `sqlserver-index-management`, `sqlserver-high-availability` |
| PostgreSQL | `postgresql-health-check`, `postgresql-query-optimization`, `postgresql-backup-recovery`, `postgresql-security`, `postgresql-replication` |

## Development Conventions

- **Platform specificity**: Never mix SQL Server and PostgreSQL guidance within the same section
- **Modern standards**: SQL Server → Always Encrypted, Always On AG; PostgreSQL → SCRAM-SHA-256, RLS, pgaudit
- **Research first**: Before building or modifying a skill, consult `research/claude/` for the relevant topic
- **Agent-executable**: Instructions must be directly actionable — T-SQL or PL/pgSQL that an agent can run verbatim

## Adding a New Skill

1. Research the topic and write findings to `research/claude/<topic>.md`
2. Create a task plan in `tasks/claude/<n>-<skill-name>.md`
3. Build the skill directory under `skills/<skill-name>/` following the structure above
4. Validate SQL queries and ensure all thresholds are specific and numeric
