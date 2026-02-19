# DBA Skills Project Overview

This directory contains a comprehensive knowledge base and a set of specialized agent skills for Database Administration (DBA) tasks, primarily focusing on **Microsoft SQL Server** and **PostgreSQL**. The project is designed to equip AI agents with the necessary expertise to perform health checks, query optimization, backup/recovery, security audits, and more.

## Directory Structure

- **`.foundry/research/`**: Contain extensive research documents on various DBA topics. These serve as the foundational knowledge for building the skills.
- **`.foundry/tasks/`**: Detailed implementation plans and task lists for each skill, following the [Agent Skills Specification](https://agentskills.io/specification).
- **Skill Directory `skills/` **: Contains skills into subdirectories, each modules containing:
    - `SKILL.md`: The core instructional file with metadata, workflows, and code snippets.
    - `examples/`: Practical scenarios and example outputs.
    - `references/`: Supplemental guides and deep-dive documentation.

## Key Files

- **`README.md`**: High-level overview of the project, supported platforms, and use cases.
- **`LICENSE`**: Project licensing information.
- **`AGENTS.md`**: (This file) Instructional context for future interactions with this project.

## Project Type: Non-Code (Knowledge Base & Agent Skills)

This is not a traditional software project with a build process. Instead, it is a structured repository of **instructional data** for AI agents.

### Usage for AI Agents

When interacting with this repository, AI agents should:
1.  **Consult Research**: Use files in `research/` subdirectories to understand the technical depth of a specific DBA topic.
2.  **Follow Skill Instructions**: Use the `SKILL.md` files in the skill directories as the primary source of truth for performing DBA tasks.
3.  **Adhere to Best Practices**: Always prioritize the latest best practices (2024-2025) documented in the research and skill files.
4.  **Reference Examples**: Use the `examples/` directories to understand how to interpret diagnostic results or apply a specific fix.

### Development Conventions

- **Skill Specification**: All skills must adhere to the [Agent Skills Specification](https://agentskills.io/specification), including YAML frontmatter and structured Markdown content.
- **Platform Specificity**: Clearly distinguish between SQL Server and PostgreSQL guidance.
- **Actionable Knowledge**: Focus on providing practical, T-SQL/SQL-heavy instructions that an agent can execute directly.
- **Modern Standards**: Emphasize 2024-2025 standards (e.g., SCRAM-SHA-256 for PostgreSQL, Always Encrypted for SQL Server).
