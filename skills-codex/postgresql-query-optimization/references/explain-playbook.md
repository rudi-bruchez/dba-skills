# EXPLAIN Playbook

## Analyze
Use execution time, row estimates vs actual, and buffer usage.

## Common Anti-Patterns
- Large sequential scans from weak predicates
- Nested loop blowups
- Sort/hash temp file pressure

## Tuning Order
Query pattern -> stats -> index design -> config adjustments.
