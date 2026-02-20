# Plan Analysis Playbook

## Common Patterns
- Key lookup explosion
- Hash spill to tempdb
- Bad cardinality estimate
- Non-SARGable predicate

## Tuning Sequence
1. Validate filter/selectivity logic.
2. Review statistics quality/freshness.
3. Evaluate index support.
4. Consider query rewrite.
5. Use hint only when justified and reversible.
