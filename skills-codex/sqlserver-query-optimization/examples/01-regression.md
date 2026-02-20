# Example: Post-Release Regression

## Prompt
A query became 5x slower after deployment.

## Expected Approach
- Compare Query Store plans by time window.
- Identify plan change root cause.
- Propose fix + rollback path.
