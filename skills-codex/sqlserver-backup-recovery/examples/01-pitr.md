# Example: Point-In-Time Recovery

## Prompt
Restore database to 10:14 before accidental delete.

## Expected Approach
- Identify valid full/diff/log chain.
- Apply logs up to recovery target.
- Validate data consistency and app readiness.
