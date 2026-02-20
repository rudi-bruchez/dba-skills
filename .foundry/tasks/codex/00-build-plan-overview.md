# Codex Skill Build Plan Overview

## Goal
Create Codex-authored DBA skills without reusing existing skill content, with strong trigger semantics and reusable references/examples.

## Execution Order
1. Finalize skill list and scope boundaries.
2. Scaffold each skill directory in `skills-codex/`.
3. Write `references/` first from research matrix.
4. Write `examples/` with realistic prompt/response patterns.
5. Write concise `SKILL.md` pointing to references/examples.
6. Validate structure/frontmatter for each skill.
7. Run prompt-trigger dry runs and refine descriptions.

## Required Artifacts Per Skill
- `SKILL.md`
- `references/*.md` (minimum 2 files)
- `examples/*.md` (minimum 3 scenarios)
- `scripts/` only if deterministic automation is needed

## Quality Gates
- Frontmatter contains only `name` and `description`.
- Description includes explicit trigger contexts and synonyms.
- SQL is platform-specific, safe, and annotated by risk.
- High-risk actions require explicit confirmation wording.
- Example outputs follow a consistent structure.

## Global Risk Controls
- Prefer read-only diagnostics first.
- Include rollback notes for each risky recommendation.
- Declare assumptions and confidence when metadata is missing.
- Do not provide destructive commands without user confirmation.

