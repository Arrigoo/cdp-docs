# Agillic CDP: Hosting and Operations

This document describes how an Agillic CDP installation is hosted and operated: the
servers, the services on each of them, how they recover from failure, how keys are
managed and rotated, how data is backed up, and how logs are collected and retained.

Every client gets a **dedicated installation**. Servers, private network, database,
backup storage area and log tenant all belong to one client. No application server
or database is shared between clients.

## At a glance

| Topic | Summary |
| --- | --- |
| Hosting provider | Hetzner Cloud |
| Data location | Helsinki, Finland (EU) |
| Isolation | Dedicated servers and a dedicated private network per client |
| Servers | 3 (API, database, background processing), plus an optional database replica |
| Operating system | Ubuntu 24.04 LTS, security updates installed automatically every day |
| Public exposure | Only the API server accepts web traffic (HTTPS). Database and background servers have no public service ports |
| Encryption in transit | HTTPS with automatically renewed Let's Encrypt certificates |
| Encryption at rest | Field-level encryption of sensitive data, with keys held in OpenBao (see [Key management](#key-management-and-rotation)) |
| Backups | Nightly encrypted database dump, stored off-server for 14 days |
| Logging | Application logs shipped to a central log store, separated per client |
| Provisioning | Fully automated (infrastructure as code). Servers can be rebuilt reproducibly |

---

## Server setup

An installation consists of three servers in the same data centre, connected by a
private network that only that client's servers are attached to.

```text
                          Internet
                             │
                             │  HTTPS (443)   HTTP (80) redirects to HTTPS
                             ▼
                  ┌──────────────────────┐
                  │      API server      │
                  │  reverse proxy + API │
                  └──────────┬───────────┘
                             │
      ═══════════════════════╪══════════════════════════  private network
                             │                             (this client only)
            ┌────────────────┼──────────────────┐
            ▼                                   ▼
 ┌──────────────────────┐            ┌──────────────────────┐
 │   Database server    │◄───────────│  Background server   │
 │ PostgreSQL, Redis,   │            │ event processing,    │
 │ OpenBao (keys)       │            │ scheduled jobs, dbt  │
 └──────────┬───────────┘            └──────────────────────┘
            │ streaming replication (optional)
            ▼
 ┌──────────────────────┐
 │ Database replica     │
 │ (optional)           │
 └──────────────────────┘
```

| Server | Purpose | Reachable from the internet |
| --- | --- | --- |
| **API server** | Terminates HTTPS and serves the CDP API and admin interface | Ports 80 and 443 only |
| **Database server** | Holds all client data: PostgreSQL database, Redis queue/cache, and the encryption key service (OpenBao) | No service ports. Reachable only over the private network |
| **Background server** | Consumes incoming events, executes actions, and runs the scheduled jobs (imports, segmentation, property assignment, data models, clean-up) | No service ports |
| **Database replica** (optional) | A continuously updated read-only copy of the database on a separate server | No service ports |

### Network and access control

- **Firewalls** in front of every server permit web traffic to the API server only.
  The database and background servers accept nothing from the internet apart from
  administrative SSH.
- **Administrative access (SSH)** is only accepted from Agillic's fixed operator IP
  addresses, and only with SSH keys. Password login is disabled on all servers.
- **Internal services** (database, queue, key service) listen on the private network
  only. That network is created for one client and no other client's servers are
  attached to it.
- **TLS**: certificates are issued by Let's Encrypt and renewed automatically. TLS 1.2
  is the minimum by default. A TLS 1.3-only configuration is available per client.

### Storage and sizing

Server sizes are chosen per client according to data volume and load, and can be
scaled up. The database is normally stored on a separate block storage volume.
That volume is independent of the server, so the database server can be replaced
without moving the data.

### How servers are built and maintained

Nothing is configured by hand. Servers, networks, firewalls and storage are created
with Terraform. Everything installed on them is configured with Ansible. Both are
kept under version control, so a server can be rebuilt with the same configuration
at any time.

- **Operating system patches**: Ubuntu's automatic security updates are enforced on
  every server and run daily. They do not reboot servers automatically. Reboots are
  planned.
- **Application releases** are rolled out by re-running the deployment. Every
  deployment recreates all containers from the pinned release version, so running
  services always match the declared configuration.
- **Secrets** (database passwords, registry and log credentials, key-service
  credentials) are never stored in plaintext. They are kept encrypted in the
  deployment repository and decrypted only on the operator machine during a
  deployment.

---

## Services per server

All services run as Docker containers, apart from the scheduled jobs, which are
systemd timers that start a short-lived container for each run.

### API server

| Service | Function | Restart policy |
| --- | --- | --- |
| Traefik | Reverse proxy. Terminates HTTPS, obtains and renews certificates, redirects HTTP to HTTPS | `always` |
| Agillic API | CDP API and admin interface | `always` |
| Grafana Alloy | Ships the API's logs to the central log store | `unless-stopped` |

### Database server

| Service | Function | Restart policy |
| --- | --- | --- |
| PostgreSQL 17 | Primary data store | `always` |
| Redis 7 | Queue and cache | `always` |
| OpenBao | Holds the key-encryption key used for encryption at rest (see [Key management](#key-management-and-rotation)) | `unless-stopped` and sealed after a restart (see below) |
| Backup job | Nightly encrypted database dump (systemd timer, 03:15) | Runs on schedule |

### Background server

| Service | Function | Restart policy |
| --- | --- | --- |
| Event consumer | Processes incoming events | `always` |
| Action consumer | Executes actions, for example sending to connected marketing systems | `always` |
| Scheduled jobs | Short-lived runs of the CDP command-line worker and dbt, started by systemd timers (see below) | Next run on schedule |
| Grafana Alloy | Ships consumer and job logs to the central log store | `unless-stopped` |

Scheduled jobs, by frequency:

| Frequency | Jobs |
| --- | --- |
| Every minute | API and CSV imports, inventory import, property assignment, segment-triggered actions, scheduled actions, AI insights queue, heartbeat |
| Every 2 minutes | Message-queue recovery |
| Every 5 minutes | Segmentation, near-real-time data models (dbt) |
| Nightly | Full property re-assignment, inventory property assignment, clean-up of expired segments, logs, customers and events, daily data models (dbt), removal of stopped containers |

### Database replica (optional)

| Service | Function | Restart policy |
| --- | --- | --- |
| PostgreSQL 17 (hot standby) | Continuously streamed read-only copy of the primary database | `always` |

### Automatic restart and recovery

- **Crashed services restart by themselves.** Containers with the `always` policy are
  restarted by Docker whenever they exit, and are started again when the server boots.
- **Docker upgrades do not interrupt service.** The Docker daemon runs with
  *live-restore*, so running containers keep running while the daemon itself is
  restarted or upgraded.
- **Scheduled jobs are self-contained.** Each run is a fresh container that is removed
  afterwards, so a failed run leaves nothing behind and the next run starts clean.
  Runs missed while a server was down are skipped, not replayed in a burst. The next
  scheduled run picks up the pending work.
- **Log shippers** use `unless-stopped`: they come back after a crash or reboot but
  stay stopped if an operator stops them deliberately.
- **The key service is fail-closed.** If field-level encryption is enabled, OpenBao
  starts **sealed** after any restart of its container or the database server, and
  it cannot hand out keys until an operator unseals it. The unseal key is never stored
  on the server. Until then the application processes refuse to start rather than run
  without encryption. Planned maintenance includes unsealing as part of the deployment
  run. After an *unplanned* reboot of the database server, the service stays down
  until an operator has unsealed it. This is deliberate: a stolen or copied server
  disk does not contain what is needed to decrypt the data.
- **Deployments restart everything.** Every deployment recreates all containers, so
  the running state always matches the declared configuration.
- **Monitoring**: both consumers and the key scheduled jobs (property assignment,
  segmentation and the nightly clean-ups) are watched by "stopped logging" alerts.
  If one goes silent for longer than its expected interval, Agillic operations are
  notified by email.

---

## Key management and rotation

### Encryption at rest

Sensitive data is encrypted at field level by the application, using envelope
encryption:

1. **Data-encryption keys (DEKs)** encrypt the data. They are stored in the database
   only in wrapped (encrypted) form.
2. **The key-encryption key (KEK)** wraps the DEKs. It lives in an OpenBao Transit
   engine on the client's database server (AES-256-GCM), is marked non-exportable,
   and never leaves OpenBao.
3. Application processes authenticate to OpenBao with a dedicated, least-privilege
   identity that can only *wrap and unwrap*. It cannot read, export or rotate the
   key. Its access tokens are valid for 1 hour, with a maximum of 4 hours.
4. The processes contact OpenBao only at start-up, to unwrap the DEKs, and during a
   DEK rotation. Normal request handling does not depend on it.
5. Every wrap and unwrap is recorded in OpenBao's audit log, together with the
   identity that made the request.

A copy of the database therefore cannot be decrypted without the KEK, and the KEK
cannot be used without the unseal key, which is held by Agillic operators and never
stored on the servers.

### Keys and credentials, and how they are rotated

| Key / credential | Protects | Stored | Rotation |
| --- | --- | --- | --- |
| TLS certificate | Traffic between users/integrations and the API | API server | Automatic. 90-day Let's Encrypt certificates, renewed well before expiry |
| Key-encryption key (KEK) | The data-encryption keys | OpenBao on the database server, non-exportable | Operator-initiated, **without downtime**. New wraps use the new key version. Earlier versions are kept so existing data stays readable |
| Data-encryption keys (DEKs) | Encrypted fields in the database | Database, wrapped by the KEK | Operator-initiated with an application command, followed by a rolling restart of the application processes |
| OpenBao unseal key | Access to the KEK after a restart | Only in Agillic's encrypted secret store, never on a server | Replaced if a compromise is suspected (re-key) |
| Application credentials to OpenBao | The application's right to wrap/unwrap | Encrypted secret store, delivered to the servers at deployment | Short-lived tokens (1 h) issued automatically. The underlying credential is replaced on demand |
| Backup encryption key pair | Database backups | Public key on the database server. Private key only in Agillic's encrypted secret store | Operator-initiated, with overlap: the previous key stays available until every backup made with it has expired (14 days) |
| Database, queue, registry, log-shipping and backup-storage passwords | Internal service access | Encrypted secret store, delivered at deployment | Changed in the secret store and applied with a deployment |
| SSH administrator key | Server administration | Agillic operators | Rotated centrally for all servers |

Rotation is triggered by operators, for example on personnel changes, on suspicion
of compromise, or on a client's request. It is not tied to a fixed calendar schedule.
Because every secret is delivered by the automated deployment, changing a credential
is an edit to the encrypted secret store followed by a deployment. Nothing has to be
changed by hand on individual servers.

---

## Backup

### Database backups

| | |
| --- | --- |
| **What** | A complete logical dump of the PostgreSQL database |
| **When** | Nightly at 03:15 (server time, UTC) |
| **Encryption** | Compressed and encrypted with a client-specific key **before it is written to disk**. The database server holds only the public half of that key, so it can create backups but cannot read any of them |
| **Where** | A Hetzner Storage Box, separate from the client's servers. Each client has its own sub-account, confined to its own directory, with no access to other clients' backups. The storage account is not reachable from the internet, only from inside Hetzner's network |
| **Transfer** | SFTP with key authentication. Uploads go to a temporary name and are renamed only when complete, so an interrupted transfer never looks like a valid backup |
| **Retention** | 14 days on the Storage Box. The two most recent dumps are also kept on the database server for fast restores |
| **Recovery point** | Up to 24 hours of data without a replica. With the optional replica, the standby copy is only seconds behind |

A compromised database server therefore cannot read any past backup, and the backup
storage alone is useless without the private key, which is held only by Agillic
operators.

### Restore

Restores are scripted. The procedure stops the application, takes a safety copy of
the current database on the database server (so the restore itself can be undone),
loads the chosen backup, and restarts the application, which then applies any
pending schema migrations. If anything fails, the application is left stopped rather
than running against a partly restored database.

### Encryption keys

The OpenBao storage holding the KEK is backed up **separately** from the database
backups. It is encrypted to an offline operator key and kept in a different location
with different access. A database backup and the key that decrypts it are never
stored together.

### Replica as a backup complement

A streaming replica on a separate server keeps a near real-time copy of the
database. It protects against loss of the primary server or its storage, and can
serve read-only queries. Promoting the replica to primary is a controlled, manual
operator action, not an automatic failover.

### Servers

Servers are not backed up as disk images. Their entire configuration is code, so a
lost server is rebuilt from scratch and the database restored onto it.

---

## Logging

### What is logged

| Source | Contents | Collected by |
| --- | --- | --- |
| API | Requests and application events | Alloy on the API server |
| Event and action consumers | Event processing and actions sent to connected systems | Alloy on the background server |
| Scheduled jobs and backup | Output of every job run, per job | systemd journal, then Alloy on the background server |
| OpenBao | Audit log of every key wrap/unwrap, with requesting identity | Kept on the database server |
| Infrastructure services (proxy, database, queue) | Service logs | Kept on the server |

System logs never contains personal data.

### Central log store

Application and job logs are shipped continuously over HTTPS to Agillic's central
log store (Grafana Loki), where operations staff use them for monitoring and
troubleshooting.

- **Separated per client.** Each client's logs go into a separate tenant, and the
  per-client monitoring view can only query that tenant.
- **Monitored.** Heartbeat alerts fire if a consumer or key job stops producing
  logs (see [Automatic restart and recovery](#automatic-restart-and-recovery)).

### Log retention

| Where | Retention |
| --- | --- |
| Central log store | Currently kept without a time limit (no automatic expiry configured yet) |
| systemd journal on each server | Persistent across reboots, capped at 500 MB per server. The oldest entries are removed first |
| Container logs on each server | Capped at 5 files of 50 MB per container (250 MB). The oldest file is removed first |
| OpenBao audit log | Kept on the database server |
| CDP application data (events, logs, segments) | Cleaned up by the nightly clean-up jobs according to the platform's data retention settings |

Local server logs are bounded by size, not age. On a busy server they cover a
shorter period than on a quiet one.

---

## Summary of optional features

| Feature | Default | Effect |
| --- | --- | --- |
| Database replica | Off | Near real-time copy of the database on a fourth server |
| Field-level encryption with OpenBao | Per client | Encryption at rest with a key held outside the database |
| TLS 1.3 only | Off (TLS 1.2 minimum) | Refuses TLS 1.2 connections |
