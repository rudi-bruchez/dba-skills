# Example: Temp File Pressure

## Prompt
Large temp files appear during reporting queries.

## Expected Approach
- Confirm sort/hash pressure in EXPLAIN.
- Reduce data processed and improve plan shape.
- Validate temp usage reduction.
