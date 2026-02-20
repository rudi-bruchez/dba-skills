# Example: Excessive Logical Reads

## Prompt
CPU is high due to one reporting query with huge reads.

## Expected Approach
- Review predicates for SARGability.
- Validate index coverage and key order.
- Measure before/after reads and CPU.
