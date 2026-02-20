# Example: PITR

## Prompt
Recover cluster to 14:22 before destructive migration.

## Expected Approach
- Confirm usable base backup and WAL range.
- Apply recovery target precisely.
- Validate data and service health.
