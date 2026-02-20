# PostgreSQL Backup Methods

## Logical
Use `pg_dump`/`pg_dumpall` for portability and granular restores.

## Physical
Use base backup + WAL archiving for full-cluster recovery and PITR.

## Selection Rule
Choose method based on RPO/RTO, data volume, and restore granularity needs.
