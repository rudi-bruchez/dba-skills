# Switchover and Failover Runbook

## Procedure
1. Confirm replica readiness and lag status.
2. Freeze risky write operations if needed.
3. Execute controlled transition.
4. Validate client routing and role states.
5. Confirm backup/archiving continuity.

## Safety
Require explicit user approval before promotion steps.
