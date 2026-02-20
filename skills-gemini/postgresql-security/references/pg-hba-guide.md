# pg-hba Guide

The `pg_hba.conf` configuration file controls client authentication based on database, user, host, and method.

## 1. Structure of `pg_hba.conf`
Each line in the file defines a specific rule for client access.
- **Type:** `local` (Unix socket), `host` (IPv4/IPv6), `hostssl` (SSL/TLS), `hostnossl` (no SSL/TLS).
- **Database:** `all`, `sameuser`, `samerole`, `replication`, or a specific database name.
- **User:** `all`, `+role_name`, or a specific username.
- **Address:** The client's IP address or hostname.
- **Method:** `trust` (no password), `scram-sha-256`, `md5`, `peer` (local user), `ident` (remote user), `cert` (SSL certificate), `ldap`, `pam`, `gssapi`.

## 2. Examples of Common Rules
```conf
# Trust local connections via Unix socket
local   all             all                                     trust

# Enforce SCRAM-SHA-256 for all IPv4 host connections
host    all             all             0.0.0.0/0               scram-sha-256

# Allow SSL-only connections for a specific database and user
hostssl mydb            myuser          192.168.1.0/24         scram-sha-256

# Allow replication for a specific role and host
host    replication     repl_user       192.168.1.100/32        scram-sha-256
```

## 3. Best Practices
- **Restrict Access:** Grant access only to the necessary databases and users from trusted hosts.
- **Enforce SSL/TLS:** Use `hostssl` for all connections to encrypt data in transit.
- **Prefer SCRAM-SHA-256:** It's more secure than `md5` or `trust`.
- **Regular Review:** Regularly audit the `pg_hba.conf` file to ensure that rules are still necessary and correct.

## References
- [PostgreSQL Documentation: pg_hba.conf](https://www.postgresql.org/docs/current/auth-pg-hba-conf.html)
- [PostgreSQL Documentation: Client Authentication](https://www.postgresql.org/docs/current/auth-methods.html)
