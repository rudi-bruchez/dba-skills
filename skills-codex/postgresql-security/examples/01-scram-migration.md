# Example: SCRAM Migration

## Prompt
Migrate from MD5 password auth to SCRAM without downtime.

## Expected Approach
- Inventory client compatibility.
- Stage credential updates and `pg_hba.conf` changes.
- Validate and keep rollback plan ready.
