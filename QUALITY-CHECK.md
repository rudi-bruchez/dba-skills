# Quality Check Standards

## Refinement Goal (skills-codex/)

Refine all skills in `skills-codex/` to improve trigger precision and reduce overlap.

For each skill:

1. Create `agents/openai.yaml` with:
   - `interface.display_name`
   - `interface.short_description`
   - `interface.default_prompt`
2. Update `SKILL.md` by adding a `## Scope Boundaries` section before `## Safety Rules`, with:
   - `### In Scope` (3 explicit bullets)
   - `### Out of Scope` (3 explicit bullets)
3. Keep YAML frontmatter in `SKILL.md` unchanged and spec-compliant:
   - only `name`
   - only `description`

## Validation Checklist

- [ ] `openai.yaml` exists for all 11 skills in `skills-codex/`.
- [ ] `SKILL.md` frontmatter keys are exactly `name`, `description`.
- [ ] Scope boundaries clearly separate adjacent domains (e.g., health-check vs query-optimization).
- [ ] No scope overlaps between skills.

## General Quality Standards (All Skill Variants)

In addition to the refined scope, all skills (including `skills/`, `skills-codex/`, and `skills-gemini/`) must adhere to these standards:

1. **Agent Skills Specification Compliance**:
   - All skills MUST conform to [Agent Skills Specification](https://agentskills.io/specification).
   - **SKILL.md frontmatter**:
     - `name`: max 64 chars, lowercase + hyphens only.
     - `description`: max 1024 chars, third person, explains what + when to use.
   - **SKILL.md body**: under 500 lines, use progressive disclosure, link details to `references/`.
   - **Directory structure**: Each skill must have `SKILL.md`, `references/`, and `examples/` subdirectories.

2. **SQL Quality**:
   - All SQL code MUST be tested and valid.
   - Use specific DMV/system view names (e.g., `sys.dm_exec_requests`, `pg_stat_activity`).
   - Include numeric thresholds and benchmarks (no vague terms like "high" or "low").
   - All SQL must be directly executable by an agent (no placeholders without context).

3. **Platform Specificity**:
   - Never mix SQL Server and PostgreSQL guidance within the same section.
   - SQL Server and PostgreSQL versions of the same skill should follow similar instructional flow.

4. **Modern Standards (2024-2025)**:
   - **PostgreSQL**: Emphasize `pg_stat_statements`, `pg_stat_activity`, and SCRAM-SHA-256. Use `EXPLAIN (ANALYZE, BUFFERS)` for optimization.
   - **SQL Server**: Prioritize `Query Store` (sys.query_store_*), modern DMVs (sys.dm_os_wait_stats), and Always Encrypted/TLS 1.3.

5. **Safety First**:
   - Every `SKILL.md` MUST have a `## Safety Rules` section.
   - Mandatory rule: "Default to read-only diagnostics before any change."
   - Mandatory rule: "Require explicit user confirmation for disruptive or irreversible actions."

6. **Reference Integrity**:
   - All links to `references/*.md` and `examples/*.md` must be valid and files must exist in their respective subdirectories.
   - Use forward slashes in all file paths (e.g., `references/health-metrics.md`).

7. **Actionable Knowledge**:
   - Focus on providing practical, T-SQL/SQL-heavy instructions that an agent can execute directly.
   - Avoid generic or vague advice.
   - Examples must be concrete scenarios with example outputs.

8. **Timeless Language**:
   - No date references in instructions (e.g., avoid "as of 2024").
   - Use "current best practices" language instead.

9. **Output Consistency**:
   - Every `SKILL.md` MUST include a `## Required Output Shape` section to standardize agent responses.
   - Standard output shape:
     - Summary of top findings or objectives.
     - Evidence used (queries/metrics/events).
     - Recommended actions ranked by impact and risk.
     - Validation plan and residual risks.

10. **Naming and Structural Consistency**:
    - **Directory Names**: Must match the `name` field in YAML frontmatter.
    - **Cross-Variant Sync**: Skill names and sets should be identical across `skills/`, `skills-codex/`, and `skills-gemini/`.
    - **Cross-Platform Parity**: PostgreSQL and SQL Server versions of the same skill (e.g., `health-check`) should follow a similar instructional flow.