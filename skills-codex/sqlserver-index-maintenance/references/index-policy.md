# Index Maintenance Policy

## Decision Principles
- Do not rebuild solely due to threshold crossing.
- Prioritize indexes tied to user-visible latency.
- Consider write overhead and fill factor side effects.

## Risk Gate
Estimate transaction log growth and runtime before major rebuilds.
