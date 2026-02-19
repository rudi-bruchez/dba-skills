# pg_hba.conf Configuration Guide

## File Format

Each line in `pg_hba.conf` matches a connection attempt and specifies the authentication method.

```
TYPE  DATABASE  USER  ADDRESS  METHOD  [OPTIONS]
```

PostgreSQL reads rules top-to-bottom and uses the **first matching rule**. Order matters.

### Field Values

**TYPE:**
| Value | Meaning |
|-------|---------|
| `local` | Unix domain socket connections (no address field) |
| `host` | TCP/IP connections (SSL or non-SSL) |
| `hostssl` | TCP/IP connections that use SSL only |
| `hostnossl` | TCP/IP connections that do not use SSL |
| `hostgssenc` | TCP/IP connections using GSSAPI encryption |

**DATABASE:**
- `all` — matches any database
- `sameuser` — database name must match the username
- `samerole` — user must be a member of a role with the same name as the database
- `replication` — physical replication connections
- `mydb,otherdb` — comma-separated specific database names
- `@file.txt` — read list from a file

**USER:**
- `all` — any user
- `username` — specific user
- `+rolename` — any member of the role (group)
- `alice,bob` — comma-separated users
- `@file.txt` — read list from a file

**ADDRESS (for host/hostssl/hostnossl):**
- `127.0.0.1/32` — IPv4 loopback
- `::1/128` — IPv6 loopback
- `10.0.0.0/8` — CIDR range
- `192.168.1.100/32` — specific host
- `0.0.0.0/0` — all IPv4 addresses
- `::/0` — all IPv6 addresses
- `hostname.example.com` — hostname (resolved at connection time)

---

## Authentication Methods

| Method | Security Level | Description |
|--------|--------------|-------------|
| `scram-sha-256` | High | Challenge-response; prevents password interception; default since PG14 |
| `md5` | Medium | Legacy; weak hash; vulnerable to replay; avoid for new setups |
| `password` | None | Cleartext password; never use over TCP/IP |
| `trust` | None | No password required; use only for unix socket + postgres role |
| `peer` | High | Validates OS username matches DB username; unix socket only |
| `ident` | Medium | Like peer but for TCP; requires identd server; rarely used |
| `cert` | Very High | Client certificate verification; requires CA + client cert |
| `gss` | High | Kerberos/GSSAPI; enterprise AD environments |
| `ldap` | Varies | Delegates to LDAP directory server |
| `pam` | Varies | Uses OS PAM modules |
| `radius` | Varies | RADIUS authentication server |
| `reject` | N/A | Always rejects; used for explicit deny rules |

---

## Recommended Configurations

### Minimal Secure Configuration

```
# TYPE  DATABASE  USER      ADDRESS          METHOD
# Local unix socket: postgres uses peer, others use scram-sha-256
local   all       postgres                   peer
local   all       all                        scram-sha-256

# IPv4 loopback
host    all       all       127.0.0.1/32     scram-sha-256

# IPv6 loopback
host    all       all       ::1/128          scram-sha-256

# Reject all remote connections
host    all       all       0.0.0.0/0        reject
```

### Production Configuration with Application Servers

```
# TYPE  DATABASE        USER            ADDRESS          METHOD
# PostgreSQL superuser via unix socket
local   all             postgres                         peer

# All local connections
local   all             all                              scram-sha-256

# Local loopback
host    all             all             127.0.0.1/32     scram-sha-256
host    all             all             ::1/128          scram-sha-256

# Streaming replication (specific subnet for standby servers)
host    replication     replicator      10.0.1.0/24      scram-sha-256

# Application tier (SSL required, specific network range)
hostssl all             app_user        10.0.2.0/24      scram-sha-256
hostssl all             app_readonly    10.0.2.0/24      scram-sha-256

# Monitoring
host    all             monitoring_user 10.0.3.10/32     scram-sha-256

# Deny everything else
host    all             all             0.0.0.0/0        reject
host    all             all             ::/0             reject
```

### Configuration with Certificate Authentication

```
# Service accounts authenticate with client certificates
hostssl all             service_account  10.0.0.0/8      cert clientcert=verify-full
```

This requires:
1. SSL CA certificate in `ssl_ca_file`
2. Client certificate signed by that CA
3. `CN` field in client cert must match the PostgreSQL username

---

## Options for Specific Methods

### scram-sha-256 with channel binding (PG14+)
```
hostssl  all  all  10.0.0.0/8  scram-sha-256 clientcert=verify-ca
```

### LDAP Authentication
```
host  all  all  10.0.0.0/8  ldap ldapserver=ldap.example.com ldapbasedn="dc=example,dc=com" ldapsearchattribute=uid
```

### RADIUS
```
host  all  all  10.0.0.0/8  radius radiusservers="radius.example.com" radiussecrets="sharedsecret"
```

---

## Managing pg_hba.conf

### Find the File Location
```sql
SHOW hba_file;
-- /etc/postgresql/16/main/pg_hba.conf  (Debian/Ubuntu)
-- /var/lib/pgsql/16/data/pg_hba.conf   (RHEL/CentOS)
```

### Reload After Changes (No Restart Needed)
```sql
-- From SQL (requires superuser)
SELECT pg_reload_conf();

-- From OS
pg_ctl reload -D /var/lib/postgresql/data
systemctl reload postgresql
```

### Verify Changes Are Active
```sql
-- View parsed rules (PostgreSQL 10+)
SELECT line_number, type, database, user_name, address, auth_method, options, error
FROM pg_hba_file_rules
ORDER BY line_number;

-- Check for parse errors
SELECT line_number, error
FROM pg_hba_file_rules
WHERE error IS NOT NULL;
```

---

## Common Mistakes and Fixes

### Mistake 1: trust on network interfaces

```
# WRONG: Any user on the network can connect as any user
host  all  all  10.0.0.0/8  trust

# CORRECT: Use scram-sha-256
host  all  all  10.0.0.0/8  scram-sha-256
```

### Mistake 2: Overly broad CIDR range

```
# RISKY: Allows any IP on the internet
host  all  all  0.0.0.0/0  scram-sha-256

# BETTER: Restrict to known networks
host  all  all  10.0.2.0/24  scram-sha-256
host  all  all  192.168.1.0/24  scram-sha-256
host  all  all  0.0.0.0/0  reject
```

### Mistake 3: md5 instead of scram-sha-256

```
# LEGACY: md5 is vulnerable to replay attacks
host  all  all  10.0.0.0/8  md5

# UPGRADE: Check that clients support scram-sha-256
# Most clients since 2019 do. Update to:
host  all  all  10.0.0.0/8  scram-sha-256
```

Client configuration: clients must set `password_encryption = scram-sha-256` for new passwords.
```sql
-- Check current password encryption setting
SHOW password_encryption;

-- Force scram-sha-256 globally
ALTER SYSTEM SET password_encryption = 'scram-sha-256';
SELECT pg_reload_conf();

-- Passwords must be reset to upgrade from md5 to scram-sha-256
-- (Existing md5 hashes are not automatically converted)
ALTER ROLE app_user PASSWORD 'same_password_forces_rehash';
```

### Mistake 4: replication entry missing

```
# Without this, pg_basebackup and standby connections will fail
# Add BEFORE the reject-all rule:
host  replication  replicator  10.0.1.0/24  scram-sha-256
```

---

## Security Best Practices Summary

1. **Use `scram-sha-256`** for all new authentication rules.
2. **Never use `trust`** for network (TCP/IP) connections.
3. **Use `hostssl`** instead of `host` for all application connections.
4. **Restrict source addresses** — use specific CIDR ranges, not `0.0.0.0/0`.
5. **End with a reject-all rule** to deny any connection not explicitly permitted.
6. **Separate replication entries** — use specific IPs for standby servers.
7. **Test changes** on a non-production instance before applying to production.
8. **Monitor failed authentication** in PostgreSQL logs: set `log_connections = on`.
