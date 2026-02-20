# Backup Strategy Matrix

## Recovery Model Alignment
- SIMPLE: full + optional diff, no log restore chain.
- FULL/BULK_LOGGED: full + diff + log backups for PITR.

## Baseline Controls
- Use `WITH CHECKSUM` on backups.
- Encrypt backup media when required.
- Store offsite copies per policy.
