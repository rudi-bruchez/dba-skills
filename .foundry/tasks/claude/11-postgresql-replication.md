# Task: Build `postgresql-replication` Skill

## Skill Metadata

```yaml
name: postgresql-replication
description: Configures and monitors PostgreSQL replication including streaming replication, logical replication, Patroni HA clusters, and connection pooling with PgBouncer. Use when setting up standby servers, monitoring replication lag, performing failovers, configuring logical replication for specific tables, or setting up PgBouncer connection pooling.
metadata:
  author: dba-skills
  version: "1.0"
  platform: PostgreSQL 13+
```

## Directory Structure

```
postgresql-replication/
├── SKILL.md
├── references/
│   ├── streaming-replication.md  # Physical streaming replication setup
│   ├── logical-replication.md    # Logical replication (publications/subscriptions)
│   └── pgbouncer-guide.md        # PgBouncer configuration and management
└── examples/
    └── examples.md               # Replication setup and troubleshooting scenarios
```

## SKILL.md Content Plan

### Sections

1. **Replication Type Selection** - Physical vs logical vs synchronous
2. **Streaming Replication Health** - Monitoring lag and sync status
3. **Standby Server Setup** - Primary and standby configuration
4. **Logical Replication** - Publications and subscriptions
5. **Failover Procedures** - Promoting a standby to primary
6. **PgBouncer** - Connection pooling setup and monitoring
7. **Patroni Overview** - HA automation framework basics

### Key System Views

**On primary:**
- `pg_stat_replication` - Connected standby status
- `pg_replication_slots` - Replication slots
- `pg_publication` - Logical publications
- `pg_subscription` - Logical subscriptions (on subscriber)
- `pg_current_wal_lsn()` - Current WAL position
- `pg_walfile_name(pg_current_wal_lsn())` - Current WAL file

**On standby:**
- `pg_stat_wal_receiver` - WAL receiver status
- `pg_is_in_recovery()` - True if standby
- `pg_last_xact_replay_timestamp()` - Last replayed transaction
- `pg_last_wal_receive_lsn()` - Last received LSN
- `pg_last_wal_replay_lsn()` - Last replayed LSN
- `pg_wal_lsn_diff(pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn())` - Replay lag bytes

**Logical replication:**
- `pg_subscription_rel` - Subscription relation status
- `pg_stat_subscription` - Subscription stats

### Key Configuration Parameters

**postgresql.conf (primary):**
```
wal_level = replica              # minimum for streaming
max_wal_senders = 10             # max standby connections
wal_keep_size = 1024             # MB of WAL to retain
hot_standby = on                 # allow reads on standby
synchronous_commit = on/off/remote_write  # sync level
synchronous_standby_names = ''   # sync standby list
```

**postgresql.conf (standby):**
```
hot_standby = on
recovery_min_apply_delay = 0     # delayed standby in ms
```

**pg_hba.conf (primary):**
```
host  replication  replicator  standby_ip/32  scram-sha-256
```

### Replication Lag Calculation

```sql
-- On primary: lag for each standby
SELECT
  client_addr,
  state,
  sent_lsn,
  write_lsn,
  flush_lsn,
  replay_lsn,
  pg_wal_lsn_diff(sent_lsn, replay_lsn) AS total_lag_bytes,
  write_lag,
  flush_lag,
  replay_lag
FROM pg_stat_replication;
```

### Failover Steps (Manual)

1. Verify primary is truly down
2. Identify most up-to-date standby: compare `pg_last_wal_replay_lsn()`
3. On chosen standby: `pg_ctl promote` or `SELECT pg_promote()`
4. Update connection strings / DNS / load balancer
5. Reconfigure other standbys to follow new primary
6. Rebuild old primary as new standby when recovered

### Logical Replication Concepts

- Publication: defines what data is published (table, operation)
- Subscription: connects to publication, receives changes
- Replication slots: track what subscriber has received
- Used for: cross-version migration, selective table replication, CDC

### Progressive Disclosure

- SKILL.md: Type selection + health monitoring + failover steps + links
- `references/streaming-replication.md`: Full setup and config walkthrough
- `references/logical-replication.md`: Publication/subscription setup
- `references/pgbouncer-guide.md`: PgBouncer modes, config, and monitoring
- `examples/examples.md`: Setup, monitoring, failover, logical replication

## References to Research

See `claude-research/postgresql-replication.md` for detailed findings.

## Acceptance Criteria

- [ ] Agent can monitor replication health and report lag
- [ ] Agent can guide streaming replication setup
- [ ] Agent can perform and guide a safe failover
- [ ] Agent can set up logical replication for specific tables
- [ ] Agent can configure and troubleshoot PgBouncer
