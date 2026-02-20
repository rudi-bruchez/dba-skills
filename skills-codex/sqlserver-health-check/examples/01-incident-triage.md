# Example: Incident Triage

## Prompt
Production SQL Server is slow since 08:30.

## Expected Approach
- Collect waits and blocking data.
- Correlate with resource pressure and recent deploy/change window.
- Return top 3 risks with immediate mitigations.
