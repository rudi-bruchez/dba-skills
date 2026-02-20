# Index Strategy

## Type Selection
- `btree`: equality/range on scalar values
- `gin`: containment/full-text/pattern-heavy searches
- `gist`: geometric/custom operator classes
- `brin`: very large append-heavy tables with locality

## Guardrail
Document write overhead and maintenance cost for each new index.
