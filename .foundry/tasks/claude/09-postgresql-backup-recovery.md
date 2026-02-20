# Task: Build `postgresql-backup-recovery` Skill

## Skill Metadata

```yaml
name: postgresql-backup-recovery
description: Manages PostgreSQL backup and recovery using pg_dump, pg_basebackup, and point-in-time recovery (PITR) with WAL archiving. Use when designing backup strategies, performing logical or physical backups, recovering from data loss, or setting up continuous archiving for PITR.
metadata:
  author: dba-skills
  version: "1.0"
  platform: PostgreSQL 13+
```

## Directory Structure

```
postgresql-backup-recovery/
├── SKILL.md
├── references/
│   ├── backup-commands.md       # pg_dump, pg_restore, pg_basebackup options
│   ├── pitr-setup.md            # WAL archiving and PITR configuration
│   └── recovery-scenarios.md    # Step-by-step recovery procedures
└── examples/
    └── examples.md              # Backup and recovery scenarios
```

## SKILL.md Content Plan

### Sections

1. **Backup Strategy Selection** - Logical vs physical, RPO/RTO considerations
2. **Logical Backups** - pg_dump and pg_restore
3. **Physical Backups** - pg_basebackup for base backups
4. **WAL Archiving & PITR** - Continuous archiving setup
5. **Point-in-Time Recovery** - Recovering to a specific timestamp
6. **Backup Verification** - Testing and validating backups
7. **Monitoring Backup Status** - Checking archive and backup health

### Key Commands and Tools

**Logical backups:**
- `pg_dump -Fc database > file.dump` - Custom format (recommended)
- `pg_dump -Fd database -f outdir/` - Directory format (parallel)
- `pg_dumpall > all.sql` - All databases including globals
- `pg_restore -d database file.dump` - Restore from custom format
- `pg_restore -l file.dump` - List contents
- `pg_restore --section=data` - Restore only data

**Physical backups:**
- `pg_basebackup -D /backup -Ft -z -P` - Tar format compressed
- `pg_basebackup -D /backup -Xs -P` - With WAL streaming

**PITR and recovery:**
- `recovery.conf` / `postgresql.conf` (PG 12+) settings
- `restore_command` - Command to retrieve archived WAL
- `recovery_target_time` - Target timestamp for PITR
- `recovery_target_action` - promote/pause/shutdown after target
- `pg_wal_replay_pause()` / `pg_wal_replay_resume()` - Pause/resume recovery
- `pg_is_in_recovery()` - Check if in recovery mode
- `pg_last_xact_replay_timestamp()` - Last replayed transaction time

**Archive monitoring:**
- `pg_stat_archiver` - WAL archiver statistics
- `pg_walfile_name(pg_current_wal_lsn())` - Current WAL position

### Backup Strategy Decision Guide

| Requirement | Approach |
|---|---|
| Single database, small size | pg_dump (custom format) |
| Multiple databases or globals | pg_dumpall |
| Fast restore needed | pg_basebackup + WAL archiving |
| PITR capability needed | pg_basebackup + WAL archiving |
| Schema-only backup | pg_dump --schema-only |
| Specific tables | pg_dump -t tablename |

### postgresql.conf Settings for WAL Archiving

```
wal_level = replica
archive_mode = on
archive_command = 'cp %p /archive/%f'
archive_timeout = 300  # Force segment switch every 5 min
```

### Progressive Disclosure

- SKILL.md: Strategy guide + quick commands + links to details
- `references/backup-commands.md`: Full pg_dump/pg_restore option reference
- `references/pitr-setup.md`: Complete PITR configuration walkthrough
- `references/recovery-scenarios.md`: Step-by-step recovery procedures
- `examples/examples.md`: Backup and recovery scenarios with commands

## References to Research

See `claude-research/postgresql-backup-recovery.md` for detailed findings.

## Acceptance Criteria

- [ ] Agent can choose appropriate backup method for given requirements
- [ ] Agent can run pg_dump with correct options for various scenarios
- [ ] Agent can configure WAL archiving for PITR
- [ ] Agent can guide a point-in-time recovery step by step
- [ ] Agent can verify backup integrity before relying on it
