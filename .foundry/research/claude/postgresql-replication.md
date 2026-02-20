# PostgreSQL Replication & High Availability

## Overview
PostgreSQL supports physical (streaming) replication and logical replication natively. High availability builds on top of replication using tools like Patroni, Repmgr, or cloud-managed services. Connection pooling (PgBouncer) is a critical companion for production HA setups.

---

## Streaming Replication (Physical)

### How It Works
1. Primary generates WAL (Write-Ahead Log)
2. WAL sender process on primary streams WAL to standby
3. WAL receiver on standby writes WAL to its `pg_wal` directory
4. Startup process applies WAL to data files
5. Standby can serve read-only queries (hot standby)

### Primary Server Configuration
```ini
# postgresql.conf on primary
wal_level = replica                  # Minimum for streaming replication
max_wal_senders = 10                 # Max simultaneous WAL sender processes
max_replication_slots = 10           # Max replication slots (if using slots)
wal_keep_size = 1024                 # MB of WAL to retain for standbys (PG13+)
# Pre-PG13: wal_keep_segments = 64

hot_standby = on                     # Allow reads on standby
synchronous_commit = on              # 'local', 'remote_write', 'remote_apply', 'on', 'off'
synchronous_standby_names = ''       # Empty = async; 'ANY 1 (standby1)' = sync

# For synchronous replication (zero data loss)
synchronous_standby_names = 'FIRST 1 (standby1, standby2)'
# or
synchronous_standby_names = 'ANY 2 (standby1, standby2, standby3)'
```

### pg_hba.conf on Primary
```
# Allow replication connections
host  replication  replicator  10.0.1.0/24    scram-sha-256
host  replication  replicator  10.0.2.0/24    scram-sha-256
```

### Creating the Replication Role
```sql
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'strong_password';
```

### Setting Up a Standby
```bash
# 1. Create base backup from primary
pg_basebackup \
    -h primary.example.com \
    -U replicator \
    -D /var/lib/postgresql/data \
    -Fp -Xs -P -R

# -R flag creates standby.signal and writes primary_conninfo to
# postgresql.auto.conf automatically

# 2. The auto.conf will contain:
# primary_conninfo = 'host=primary.example.com port=5432 user=replicator ...'
# primary_slot_name = ''  (if using slots)

# 3. Verify standby.signal exists
ls /var/lib/postgresql/data/standby.signal

# 4. Start standby
pg_ctl start -D /var/lib/postgresql/data
```

### Standby Configuration (postgresql.conf)
```ini
# These are set in postgresql.auto.conf by -R flag, but can override:
primary_conninfo = 'host=primary.example.com port=5432 user=replicator password=pw application_name=standby1'
primary_slot_name = 'standby1_slot'  # If using replication slots

hot_standby = on                     # Allow read-only queries
hot_standby_feedback = on            # Prevent query cancellation on standby
max_standby_streaming_delay = 30s    # Cancel standby queries after this lag
max_standby_archive_delay = 90s      # Delay for archive recovery

# Recovery settings (PG12+, previously in recovery.conf)
recovery_min_apply_delay = 0         # Delayed standby: '1h' for 1-hour lag
```

---

## Replication Slots

### Why Use Replication Slots
- Primary retains WAL until standby has consumed it
- No data loss if standby falls behind and disconnects
- **Risk**: WAL accumulates indefinitely if standby disconnects — monitor disk space!

### Managing Replication Slots
```sql
-- Create a physical replication slot
SELECT pg_create_physical_replication_slot('standby1_slot');

-- Create a logical replication slot
SELECT pg_create_logical_replication_slot('my_logical_slot', 'pgoutput');

-- View all slots
SELECT slot_name,
       slot_type,
       active,
       active_pid,
       restart_lsn,
       confirmed_flush_lsn,
       pg_size_pretty(
           pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)
       ) AS retained_wal,
       wal_status,        -- PG13+: 'reserved', 'extended', 'unreserved', 'lost'
       safe_wal_size      -- PG13+: bytes of WAL before losing the slot
FROM pg_replication_slots;

-- Drop an inactive slot (CAREFUL: data loss possible if standby reconnects)
SELECT pg_drop_replication_slot('standby1_slot');

-- Alert: slots retaining > 10GB of WAL
SELECT slot_name,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_wal
FROM pg_replication_slots
WHERE pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) > 10 * 1024^3;
```

---

## Monitoring Replication

### On Primary — Replication Status
```sql
SELECT client_addr,
       application_name,
       state,               -- 'streaming', 'catchup', 'backup', 'startup'
       sent_lsn,
       write_lsn,
       flush_lsn,
       replay_lsn,
       write_lag,           -- Time primary waited for standby to write WAL
       flush_lag,           -- Time primary waited for standby to flush WAL
       replay_lag,          -- Time primary waited for standby to replay WAL
       sync_state,          -- 'async', 'potential', 'sync', 'quorum'
       reply_time
FROM pg_stat_replication;
```

### On Standby — Recovery Status
```sql
-- Is this server in recovery?
SELECT pg_is_in_recovery();

-- Current recovery LSN and lag
SELECT pg_last_wal_receive_lsn(),
       pg_last_wal_replay_lsn(),
       pg_wal_lsn_diff(pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn()) AS replay_lag_bytes,
       now() - pg_last_xact_replay_timestamp() AS replay_lag_time;

-- Check if receiving WAL
SELECT pg_last_wal_receive_lsn() = pg_last_wal_replay_lsn() AS in_sync;
```

### Replication Lag Alert Query
```sql
-- On primary: lag in bytes per standby
SELECT application_name,
       client_addr,
       state,
       pg_wal_lsn_diff(sent_lsn, replay_lsn) AS replay_lag_bytes,
       pg_size_pretty(pg_wal_lsn_diff(sent_lsn, replay_lsn)) AS replay_lag_size,
       replay_lag AS replay_lag_time
FROM pg_stat_replication
ORDER BY replay_lag_bytes DESC;
```

---

## Logical Replication

### How It Works
- Replicates individual tables, not entire cluster
- Works across different PostgreSQL major versions
- Publisher sends row-level changes; subscriber applies them
- Useful for: migrations, selective replication, cross-version upgrades

### Setting Up Logical Replication

**On Publisher (Source):**
```ini
# postgresql.conf
wal_level = logical              # Required for logical replication
max_replication_slots = 10
max_wal_senders = 10
```

```sql
-- Create publication (all tables or specific tables)
CREATE PUBLICATION mypub FOR ALL TABLES;
-- Or specific tables:
CREATE PUBLICATION mypub FOR TABLE orders, customers, products;
-- Or specific operations:
CREATE PUBLICATION mypub FOR TABLE orders
    WITH (publish = 'insert, update, delete');

-- View publications
SELECT * FROM pg_publication;
SELECT * FROM pg_publication_tables;
```

**On Subscriber (Target):**
```sql
-- Create subscription (target tables must already exist)
CREATE SUBSCRIPTION mysub
    CONNECTION 'host=publisher.example.com dbname=mydb user=repuser password=pw'
    PUBLICATION mypub;

-- With slot options
CREATE SUBSCRIPTION mysub
    CONNECTION 'host=publisher.example.com dbname=mydb user=repuser'
    PUBLICATION mypub
    WITH (copy_data = true,    -- Copy existing data initially
          create_slot = true,  -- Create slot on publisher
          enabled = true);     -- Start immediately

-- View subscriptions
SELECT * FROM pg_subscription;
SELECT * FROM pg_stat_subscription;

-- View subscription worker status
SELECT subname, pid, received_lsn, last_msg_send_time,
       last_msg_receipt_time, latest_end_lsn, latest_end_time
FROM pg_stat_subscription;
```

### Logical Replication Conflicts
```sql
-- View replication origin tracking
SELECT * FROM pg_replication_origin;
SELECT * FROM pg_replication_origin_status;

-- Skip a conflicting transaction (use carefully)
SELECT pg_replication_origin_advance('pg_16384', '0/12345678'::pg_lsn);
```

---

## Failover & Promotion

### Manual Promotion of Standby to Primary
```bash
# Method 1: pg_ctl promote
pg_ctl promote -D /var/lib/postgresql/data

# Method 2: Create promote trigger file (if trigger_file set)
touch /tmp/postgresql.trigger.5432

# Method 3: SQL (PG12+)
-- Connect to standby:
SELECT pg_promote(wait => true, wait_seconds => 60);
```

### Post-Failover Steps
```sql
-- 1. Verify the new primary is writable
SELECT pg_is_in_recovery();  -- Should return false

-- 2. Update application connection strings / VIP

-- 3. Reconfigure old primary as new standby (requires pg_rewind or fresh basebackup)

-- Rewind old primary to new primary's timeline
-- (Must have wal_log_hints=on or checksum enabled on old primary)
pg_rewind --target-pgdata=/var/lib/postgresql/data \
          --source-server="host=new-primary.example.com user=replicator"

-- 4. Create standby.signal on rewound instance
touch /var/lib/postgresql/data/standby.signal

-- 5. Update primary_conninfo to point to new primary
```

---

## Patroni — Automated HA

### Architecture
- **Patroni**: Python daemon that manages PostgreSQL HA with a DCS (etcd, Consul, ZooKeeper, or Kubernetes)
- **DCS**: Distributed Configuration Store for leader election and cluster state
- **VIP / HAProxy / DNS**: Route application traffic to primary

### Patroni Key Concepts
```yaml
# /etc/patroni/patroni.yml (simplified)
scope: postgres-cluster
namespace: /db/
name: node1

restapi:
  listen: 0.0.0.0:8008
  connect_address: node1.example.com:8008

etcd3:
  hosts: etcd1:2379,etcd2:2379,etcd3:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576    # 1MB max lag for failover
    postgresql:
      use_pg_rewind: true
      parameters:
        wal_level: replica
        hot_standby: "on"
        max_wal_senders: 10
        max_replication_slots: 10

postgresql:
  listen: 0.0.0.0:5432
  connect_address: node1.example.com:5432
  data_dir: /var/lib/postgresql/data
  pgpass: /tmp/pgpass
  authentication:
    replication:
      username: replicator
      password: reppass
    superuser:
      username: postgres
      password: superpass
```

### Patroni CLI Commands
```bash
# Cluster status
patronictl -c /etc/patroni/patroni.yml list

# Manual switchover (graceful)
patronictl -c /etc/patroni/patroni.yml switchover --master node1 --candidate node2

# Failover (immediate, no data loss guarantee)
patronictl -c /etc/patroni/patroni.yml failover postgres-cluster --master node1

# Restart a node
patronictl -c /etc/patroni/patroni.yml restart postgres-cluster node1

# Reload Patroni config
patronictl -c /etc/patroni/patroni.yml reload postgres-cluster

# Show history of leadership changes
patronictl -c /etc/patroni/patroni.yml history postgres-cluster

# Query current primary via REST API
curl http://node1:8008/primary     # 200 = this is primary, 503 = not primary
curl http://node1:8008/replica     # 200 = this is replica
curl http://node1:8008/health      # Cluster health
```

### Patroni REST API Endpoints
| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/` | GET | Node status |
| `/primary` | GET | 200 if primary |
| `/replica` | GET | 200 if replica |
| `/health` | GET | General health |
| `/patroni` | GET | Detailed Patroni state |
| `/config` | GET/PATCH | View/update config |
| `/switchover` | POST | Trigger switchover |
| `/failover` | POST | Trigger failover |

---

## PgBouncer — Connection Pooling

### Why PgBouncer
- PostgreSQL forks a new process per connection (~5-10MB each)
- PgBouncer maintains a pool and multiplexes application connections
- Reduces connection overhead, allows higher application concurrency
- Handles connection storms during failover

### Pooling Modes
| Mode | Description | Latency | Compatibility |
|------|-------------|---------|---------------|
| `session` | One server connection per client session | None | Full |
| `transaction` | Server connection held only during transaction | Low | Most apps |
| `statement` | Server connection held only during statement | Minimal | Limited |

**Production recommendation**: `transaction` pooling.

### pgbouncer.ini Configuration
```ini
[databases]
mydb = host=localhost port=5432 dbname=mydb
# With failover awareness:
mydb = host=vip.example.com port=5432 dbname=mydb

[pgbouncer]
listen_port = 6432
listen_addr = *
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt

# Pool settings
pool_mode = transaction
max_client_conn = 1000          # Max application connections
default_pool_size = 20          # Server connections per database/user pair
min_pool_size = 5               # Keep minimum connections alive
reserve_pool_size = 5           # Extra connections for high load
reserve_pool_timeout = 3        # Seconds before using reserve pool

# Timeouts
server_idle_timeout = 600       # Close idle server connections after 10 min
client_idle_timeout = 0         # Don't disconnect idle clients
server_connect_timeout = 15
server_login_retry = 15
query_wait_timeout = 120        # Kill client if waits > 2 min for server

# Logging
log_connections = 0
log_disconnections = 0
log_pooler_errors = 1
stats_period = 60
```

### PgBouncer userlist.txt
```
# Format: "username" "md5hash_or_scram_verifier"
# Generate with: psql -c "SELECT usename, passwd FROM pg_shadow WHERE usename = 'myuser';"
"myuser" "SCRAM-SHA-256$4096:salt_base64=$stored_key_base64:$server_key_base64"
```

### PgBouncer Monitoring
```sql
-- Connect to PgBouncer admin console
psql -h localhost -p 6432 -U pgbouncer pgbouncer

-- Pool status
SHOW POOLS;
-- cl_active: clients waiting for server
-- cl_waiting: clients in queue waiting for connection
-- sv_active: server connections in use
-- sv_idle: available server connections

-- Detailed stats
SHOW STATS;
SHOW CLIENTS;
SHOW SERVERS;
SHOW CONFIG;

-- Reload config without restart
RELOAD;

-- Pause all connections (for maintenance)
PAUSE mydb;
RESUME mydb;
```

---

## Repmgr — Alternative HA Tool

```bash
# Initialize primary
repmgr -h localhost -U repmgr -d repmgr primary register

# Clone standby
repmgr -h primary -U repmgr -d repmgr standby clone -F

# Register standby
repmgr -h localhost -U repmgr -d repmgr standby register

# Show cluster status
repmgr -h localhost -U repmgr -d repmgr cluster show

# Manual switchover
repmgr standby switchover -f /etc/repmgr.conf

# Automatic failover daemon
repmgrd -f /etc/repmgr.conf
```

---

## Delayed Replication (Disaster Recovery)

```ini
# On standby: postgresql.conf or postgresql.auto.conf
recovery_min_apply_delay = '1h'    # Standby lags 1 hour behind primary
# This gives 1 hour to catch accidental DROP TABLE before it replicates
```

```sql
-- Check current delay
SELECT now() - pg_last_xact_replay_timestamp() AS current_lag;
```

---

## Key Thresholds & Alerts

| Metric | Warning | Critical |
|--------|---------|---------|
| Replication lag (bytes) | > 50MB | > 500MB |
| Replication lag (time) | > 10s | > 60s |
| Replication slot retained WAL | > 1GB | > 10GB |
| pg_wal directory size | > 2GB | > 10GB |
| Standby last replay timestamp | > 30s ago | > 5 min ago |
| WAL senders active | 0 standbys | N/A |

---

## Common Replication Issues & Solutions

| Issue | Diagnostic | Solution |
|-------|-----------|---------|
| Standby falling behind | `pg_stat_replication.replay_lag` | Increase `wal_keep_size`, check standby I/O |
| Replication slot not advancing | `pg_replication_slots.restart_lsn` | Check standby health; drop dead slots |
| pg_wal filling disk | `du -sh pg_wal/` | Drop inactive slots, increase `checkpoint_completion_target` |
| Standby query cancelled | `ERROR: canceling statement due to conflict` | Set `hot_standby_feedback=on`, increase delays |
| Split-brain after failover | Two primaries accepting writes | Always use fencing in automated failover |
| Logical replication conflict | Subscription error in logs | Identify conflict, skip or fix conflicting row |
| WAL sender slot not releasing | Active pid = NULL, slot active | Drop slot if standby is permanently gone |
