# Query Store Workflow

## Candidate Selection
Use Query Store to rank by duration, CPU, and logical reads.

## Regression Detection
Compare current interval vs known-good interval for the same query hash.

## Validation Metrics
- Duration (p50/p95)
- CPU time
- Logical reads
- Spills / memory grants

## Guardrail
Do not force plans permanently without clear review criteria.
