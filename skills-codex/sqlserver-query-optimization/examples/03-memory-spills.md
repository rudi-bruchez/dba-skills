# Example: Spill-Heavy Query

## Prompt
Sort/hash spills are saturating tempdb.

## Expected Approach
- Identify underestimation and grant mismatch.
- Improve plan shape or row estimates.
- Validate spill elimination and runtime gain.
