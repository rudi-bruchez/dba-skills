# Task: Build `sqlserver-high-availability` Skill

## Skill Metadata

```yaml
name: sqlserver-high-availability
description: Configures and troubleshoots SQL Server high availability and disaster recovery solutions including Always On Availability Groups, Failover Cluster Instances, log shipping, and database mirroring. Use when setting up HA/DR, monitoring replication health, performing planned failovers, or troubleshooting synchronization issues.
metadata:
  author: dba-skills
  version: "1.0"
  platform: SQL Server 2016+
```

## Directory Structure

```
sqlserver-high-availability/
├── SKILL.md
├── references/
│   ├── always-on-reference.md   # Always On AG setup and management
│   ├── failover-procedures.md   # Planned and unplanned failover steps
│   └── monitoring-queries.md    # HA health monitoring queries
└── examples/
    └── examples.md              # HA setup and troubleshooting scenarios
```

## SKILL.md Content Plan

### Sections

1. **HA Technology Comparison** - When to use AG vs FCI vs log shipping
2. **Always On Health Check** - Monitoring AG synchronization state
3. **Performing Failover** - Planned vs forced failover procedures
4. **Troubleshooting Sync Issues** - Diagnosing lag and hardening failures
5. **Log Shipping** - Setup and monitoring
6. **Readable Secondaries** - Offloading reads to secondary replicas
7. **Listener Configuration** - Connection routing and client setup

### Key DMVs and Catalog Views

- `sys.availability_groups` - AG definition
- `sys.availability_replicas` - Replica configuration
- `sys.dm_hadr_availability_group_states` - AG health state
- `sys.dm_hadr_availability_replica_states` - Replica sync state
- `sys.dm_hadr_database_replica_states` - Per-database replica state
- `sys.availability_databases_cluster` - Cluster-level DB info
- `sys.dm_hadr_database_replica_cluster_states` - Cluster DB states
- `sys.databases` - `is_hadr_suspended` flag
- `msdb.dbo.log_shipping_primary_databases` - Log shipping primaries
- `msdb.dbo.log_shipping_secondary_databases` - Log shipping secondaries
- `msdb.dbo.log_shipping_monitor_primary` - Log shipping monitor
- `sys.dm_os_cluster_nodes` - FCI cluster nodes

### Key Commands

- `ALTER AVAILABILITY GROUP ... FAILOVER` - Planned failover
- `ALTER AVAILABILITY GROUP ... FORCE_FAILOVER_ALLOW_DATA_LOSS` - Emergency
- `ALTER DATABASE ... SET HADR SUSPEND/RESUME` - Suspend/resume sync
- `ALTER AVAILABILITY GROUP ... JOIN` - Join replica to AG
- `ALTER DATABASE ... SET HADR AVAILABILITY GROUP` - Add DB to AG

### HA Health Thresholds

| Metric | Healthy | Warning | Critical |
|---|---|---|---|
| Synchronization state | SYNCHRONIZED | SYNCHRONIZING | NOT SYNCHRONIZING |
| Redo queue size | < 100 MB | 100 MB – 1 GB | > 1 GB |
| Log send queue | < 50 MB | 50–500 MB | > 500 MB |
| Estimated data loss | 0 | < 1 min | > 1 min |
| Log shipping latency | < 5 min | 5–30 min | > 30 min |

### Failover Decision Tree

1. Is it a planned failover? → ALTER AVAILABILITY GROUP FAILOVER
2. Primary unavailable, secondary SYNCHRONIZED? → Manual failover
3. Primary unavailable, secondary SYNCHRONIZING? → Forced failover (data loss risk)
4. All replicas unavailable? → Restore from backup

### Progressive Disclosure

- SKILL.md: Technology comparison + health check + failover decision
- `references/always-on-reference.md`: Complete AG configuration guide
- `references/failover-procedures.md`: Step-by-step failover procedures
- `references/monitoring-queries.md`: All HA monitoring queries
- `examples/examples.md`: Setup, monitoring, and troubleshooting scenarios

## References to Research

See `claude-research/sqlserver-high-availability.md` for detailed findings.

## Acceptance Criteria

- [ ] Agent can assess current AG health and report issues
- [ ] Agent can guide a planned failover safely
- [ ] Agent can troubleshoot synchronization lag
- [ ] Agent can recommend appropriate HA technology for given requirements
- [ ] Examples cover planned failover, forced failover, troubleshooting sync
