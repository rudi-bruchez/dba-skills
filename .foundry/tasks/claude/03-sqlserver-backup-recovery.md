# Task: Build `sqlserver-backup-recovery` Skill

## Skill Metadata

```yaml
name: sqlserver-backup-recovery
description: Guides SQL Server backup strategy design and restore operations including full, differential, and log backups, point-in-time recovery, and backup verification. Use when designing a backup strategy, troubleshooting backup failures, performing database restores, or validating backup integrity.
metadata:
  author: dba-skills
  version: "1.0"
  platform: SQL Server 2016+
```

## Directory Structure

```
sqlserver-backup-recovery/
├── SKILL.md
├── references/
│   ├── backup-types.md          # Full/diff/log/filegroup backup details
│   ├── restore-scenarios.md     # Step-by-step restore procedures
│   └── backup-monitoring.md     # Monitoring backup jobs and history
└── examples/
    └── examples.md              # Backup strategy examples and restore scenarios
```

## SKILL.md Content Plan

### Sections

1. **Recovery Models** - Simple, Bulk-Logged, Full — when to use each
2. **Backup Strategy Design** - RTO/RPO-driven strategy selection
3. **Performing Backups** - T-SQL for full, differential, log backups
4. **Backup Verification** - RESTORE VERIFYONLY, RESTORE HEADERONLY
5. **Point-in-Time Recovery** - Log chain, restoring to a specific time
6. **Recovery Procedures** - Step-by-step restore workflows
7. **Monitoring Backup History** - Checking backup status and gaps

### Key T-SQL Commands and Tables

- `BACKUP DATABASE ... TO DISK` - Full backup
- `BACKUP DATABASE ... WITH DIFFERENTIAL` - Differential backup
- `BACKUP LOG ... TO DISK` - Transaction log backup
- `RESTORE VERIFYONLY` - Verify backup readability
- `RESTORE HEADERONLY` - Read backup header info
- `RESTORE FILELISTONLY` - List files in backup set
- `RESTORE DATABASE ... WITH NORECOVERY` - Restore with more logs to follow
- `RESTORE DATABASE ... WITH RECOVERY` - Final restore step
- `RESTORE DATABASE ... WITH STOPAT` - Point-in-time recovery
- `msdb.dbo.backupset` - Backup history
- `msdb.dbo.backupmediafamily` - Backup file locations
- `msdb.dbo.restorehistory` - Restore history
- `sys.databases` - Recovery model, state

### Key Concepts

- Log chain integrity (broken chain = no PITR)
- Copy-only backups (don't break differential chain)
- Tail-log backup before restore
- NORECOVERY vs RECOVERY vs STANDBY
- Backup compression and encryption
- Striped backups for large databases
- Testing restores regularly

### Recovery Scenarios to Cover

1. Full database restore (simple recovery model)
2. Point-in-time restore (full recovery model)
3. Restore to different server/instance
4. Restore single table from backup (partial/piecemeal)
5. Emergency: database in SUSPECT state

### Progressive Disclosure

- SKILL.md: Strategy framework + quick reference + links to details
- `references/backup-types.md`: Detailed backup type documentation
- `references/restore-scenarios.md`: Full step-by-step restore procedures
- `references/backup-monitoring.md`: Monitoring queries and alerting
- `examples/examples.md`: 5 backup/restore scenarios

## References to Research

See `claude-research/sqlserver-backup-recovery.md` for detailed findings.

## Acceptance Criteria

- [ ] Agent can design an appropriate backup strategy given RTO/RPO requirements
- [ ] Agent can write correct T-SQL for all backup types
- [ ] Agent can guide a complete point-in-time recovery
- [ ] Agent can detect gaps in backup history
- [ ] Examples cover all major recovery scenarios
