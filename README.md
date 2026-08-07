# Patroni + PostgreSQL + pgpool-II High Availability Cluster
## A Complete Deployment and Operations Guide (Ansible + Manual)

Welcome! This repository builds a **production-grade, 3-node high-availability PostgreSQL 16 cluster** on CentOS Stream 9 (RHEL-family) VMs using:

- **Percona Distribution for PostgreSQL 16** — the database engine
- **Patroni** — the "brain" that watches the cluster and fails over automatically
- **etcd** — the distributed consensus store that holds the leader lock
- **pgpool-II 4.5 + Watchdog** — the "traffic controller" that gives applications one Virtual IP (VIP)
- **pgBackRest** — backup & point-in-time recovery
- **PMM (Percona Monitoring & Management)** — dashboards and alerts

You can deploy everything **automatically with Ansible** (recommended), or **step-by-step by hand** (great for learning exactly what is happening under the hood). Both paths are documented below.

---

## Table of Contents

1. [Introduction — The Problem Space](#1-introduction--the-problem-space)
2. [Architecture Overview — How the Pieces Fit Together](#2-architecture-overview--how-the-pieces-fit-together)
3. [High Availability Concepts — Building the Mental Model](#3-high-availability-concepts--building-the-mental-model)
4. [Patroni Internals — How the Brain Works](#4-patroni-internals--how-the-brain-works)
5. [pgpool-II — The Traffic Controller](#5-pgpool-ii--the-traffic-controller)
6. [Server Planning — Before You Install Anything](#6-server-planning--before-you-install-anything)
7. [What's Inside This Repository](#7-whats-inside-this-repository)
8. [Deployment Method A — Ansible (Automated)](#8-deployment-method-a--ansible-automated)
9. [Deployment Method B — Manual (No Ansible)](#9-deployment-method-b--manual-no-ansible)
10. [Operations Guide — Daily Commands](#10-operations-guide--daily-commands)
11. [Failure Testing — Learning by Breaking Things](#11-failure-testing--learning-by-breaking-things)
12. [Troubleshooting Common Issues](#12-troubleshooting-common-issues)
13. [Security Notes — Change These Before Production](#13-security-notes--change-these-before-production)
14. [Production Recommendations](#14-production-recommendations)
15. [External References](#15-external-references)

---

## 1. Introduction — The Problem Space

### Who Is This Guide For?

- System administrators who know PostgreSQL basics but have **never built a PostgreSQL HA cluster**
- Developers who understand `psql` but have **never heard of Patroni, etcd, or pgpool-II**
- Anyone who wants to deploy PostgreSQL HA **automatically with Ansible** *or* **by hand**
- Novice database users who want to **understand** what they are deploying, not just copy-paste commands

**Assumed knowledge:** basic Linux (shell, `systemctl`, editing files with `sudo`), basic PostgreSQL (creating tables, running `psql`). Everything about HA, Patroni, etcd, and pgpool-II is explained from zero.

### The Passwords You Will Need

Generate these **before** you start — you will paste them into several config files, and **they must be identical on all nodes**:

| Password | Purpose |
|----------|---------|
| PostgreSQL superuser (`postgres`) password | Admin access to the database |
| PostgreSQL replication (`replicator`) password | Streaming replication between nodes |
| `pgpool` monitoring user password | pgpool-II health checks |
| Patroni REST API / `admin` user password | Cluster management |
| PCP (`pgpool_pcp`) password | pgpool-II control protocol |
| PMM admin password | Monitoring web UI |

> ⚠️ This repository ships with **default example passwords** (e.g. `qaz123`, `replPasswd`) so a novice can deploy immediately. **You MUST change them before any production use** — see [Security Notes](#13-security-notes--change-these-before-production).

### A Note on Hosts

All examples use **db1/db2/db3** with IPs `192.168.122.150–152`, a backup node at `.153`, and the Virtual IP `192.168.122.200`, on **CentOS Stream 9 / RHEL-family** with `dnf`. The concepts and steps are identical on Debian/Ubuntu — only the package manager differs (`apt` instead of `dnf`).

### What Problem Does Patroni Solve?

PostgreSQL is a powerful relational database, but out of the box it has a fundamental limitation: **it does not automatically fail over when the primary server dies.**

If you run a single PostgreSQL instance and it crashes (hardware failure, kernel panic, OOM killer, network partition), your application goes down until a human intervenes. You must manually promote a replica, update DNS or connection strings, and hope nothing breaks in the process.

**Patroni solves this by adding:**

- **Automatic leader election** using a distributed consensus store (etcd)
- **Automated failover** when the primary becomes unreachable
- **Controlled switchover** for planned maintenance
- **Replication management** — Patroni handles `pg_basebackup`, replication slots, and `pg_rewind` automatically
- **A REST API** for monitoring and programmatic control

### Why PostgreSQL Alone Is Not Enough for Automatic Failover

PostgreSQL streaming replication is asynchronous by default. The replica receives Write-Ahead Log (WAL) records from the primary and replays them. But PostgreSQL has **no built-in mechanism** to:

- Decide **which** replica should become primary when the current one fails
- Coordinate that decision across multiple replicas to prevent **"split brain"** (two primaries accepting writes)
- Notify applications that the primary has moved
- **Fence** the old primary so it cannot accept writes after losing leadership

Patroni provides all of this by layering a consensus-driven state machine on top of PostgreSQL's native replication.

### The Cast of Characters — Component Roles at a Glance

| Component | Role | What It Does NOT Do |
|-----------|------|---------------------|
| **PostgreSQL** | The actual database engine — stores data, processes queries, streams WAL | Does not decide who is primary; does not fail over automatically |
| **Patroni** | Cluster manager — leader election, failover, replication orchestration, configuration management | Does not route client connections; does not pool connections |
| **etcd** | Distributed consensus store — holds leader lock, cluster state, configuration | Does not manage PostgreSQL; does not understand SQL |
| **pgpool-II** | Connection middleware — routing, pooling, read balancing, Virtual IP (VIP) management | Does not manage PostgreSQL replication; does not elect leaders |
| **pgBackRest** | Backup & restore engine — full/incremental backups, archiving, PITR | Does not manage HA; does not route traffic |
| **PMM** | Monitoring — dashboards, alerting, query analytics (pg_stat_monitor) | Does not manage HA; is not required for failover |

---

## 2. Architecture Overview — How the Pieces Fit Together

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           APPLICATION LAYER                                 │
│  Your application connects to: pgpool-II (VIP: 192.168.122.200:9999)        │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         pgpool-II CLUSTER (Watchdog)                        │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                      │
│  │  pgpool-II  │    │  pgpool-II  │    │  pgpool-II  │   ← VIP floats here  │
│  │   (db1)     │◄───│   (db2)     │◄───│   (db3)     │   ← quorum = 2/3     │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘                      │
│         │                  │                  │                             │
│         └──────────────────┼──────────────────┘                             │
│                            ▼                                                │
│              ┌─────────────────────────────────┐                            │
│              │   Health Checks (pg_isready)    │                            │
│              │   Streaming Replication Checks  │                            │
│              └─────────────────────────────────┘                            │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
              ┌─────────────────────┼─────────────────────┐
              ▼                     ▼                     ▼
┌─────────────────────┐ ┌─────────────────────┐ ┌─────────────────────┐
│    POSTGRESQL       │ │    POSTGRESQL       │ │    POSTGRESQL       │
│     PRIMARY         │ │     REPLICA         │ │     REPLICA         │
│    (db1:5432)       │ │    (db2:5432)       │ │    (db3:5432)       │
│                     │ │                     │ │                     │
│  ┌───────────────┐  │ │  ┌───────────────┐  │ │  ┌───────────────┐  │
│  │   Patroni     │  │ │  │   Patroni     │  │ │  │   Patroni     │  │
│  │   (Leader)    │  │ │  │   (Follower)  │  │ │  │   (Follower)  │  │
│  └───────┬───────┘  │ │  └───────┬───────┘  │ │  └───────┬───────┘  │
└──────────│──────────┘ └──────────│──────────┘ └──────────│──────────┘
           │                       │                       │
           └───────────────────────┼───────────────────────┘
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           etcd CLUSTER (3 nodes)                            │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐                                      │
│  │  etcd   │  │  etcd   │  │  etcd   │   ← Raft consensus: leader lock      │
│  │ (db1)   │  │ (db2)   │  │ (db3)   │   ← cluster state, config, TTL       │
│  └────┬────┘  └────┬────┘  └────┬────┘   ← quorum = 2/3                     │
│       │            │            │                                           │
│       └────────────┴────────────┘                                           │
└─────────────────────────────────────────────────────────────────────────────┘
```

Plus a **backup/monitoring node** (db-backup: `.153`):

- **pgBackRest** — receives archived WAL + makes full backups of the cluster
- **PMM Server** — the web dashboard every DBA loves (runs as a Docker container)

### Data Flow Summary

1. **Application connects to the pgpool-II VIP** (`192.168.122.200:9999`) — it never talks to individual nodes
2. pgpool-II **routes writes** to the current PostgreSQL primary (detected via streaming replication checks)
3. pgpool-II **load-balances reads** across replicas (optional, `load_balance_mode = on`)
4. **Patroni** on each PostgreSQL node watches etcd for leadership changes
5. **etcd** holds the leader lock — only one Patroni can hold it at a time
6. When the primary fails, Patroni on a replica **acquires the lock and promotes** PostgreSQL
7. pgpool-II detects the new primary via health checks and **routes traffic there**
8. The **VIP moves** to the pgpool-II node that is currently the watchdog leader (independent of the PostgreSQL primary)
9. **pgBackRest** keeps archiving WAL; **PMM** keeps graphing everything

### Network Endpoints

| Service | Port | Source → Destination | Purpose |
|---------|------|----------------------|---------|
| SSH | 22 | Admin → All nodes | Remote administration |
| **pgpool-II (VIP)** | **9999** | App servers → VIP | **Client connection endpoint** |
| PCP (pgpool control) | 9898 | Admin → All nodes | pcp_* admin commands |
| Watchdog heartbeat | 9000 | All nodes ↔ All nodes | pgpool-II watchdog inter-node checks |
| PostgreSQL | 5432 | All nodes ↔ All nodes | Replication + pgpool health checks |
| Patroni REST API | 8008 | All nodes ↔ All nodes | Health checks, `patronictl` |
| etcd client | 2379 | All nodes ↔ All nodes | Patroni ↔ etcd communication |
| etcd peer | 2380 | All nodes ↔ All nodes | etcd Raft (cluster) communication |
| pgBackRest | 22 (SSH) | PG nodes → backup node | WAL archiving / backups |
| PMM Server | 443 | Admin / PG nodes → backup node | Monitoring web UI (HTTPS) |

> 🔥 **Firewall rule:** Allow these ports **only between cluster nodes** (source = cluster subnet). Do not expose etcd, the Patroni REST API, or PostgreSQL directly to application networks — **only the pgpool-II VIP**.

---

## 3. High Availability Concepts — Building the Mental Model

If you have never worked with database HA, read this section carefully. These concepts are the foundation for everything that follows.

### Primary and Replica

- **Primary** (formerly "master"): The single PostgreSQL instance that accepts **reads AND writes**. It generates WAL (Write-Ahead Log) records for every change.
- **Replica** (formerly "slave" / "standby"): A PostgreSQL instance that **receives WAL from the primary and replays it**. It accepts reads ONLY (when `hot_standby = on`).

```
Primary (read/write) ──WAL stream──► Replica (read-only)
       │                                   │
       ▼                                   ▼
  WAL generated                      WAL replayed
```

### Streaming Replication

PostgreSQL's native replication mechanism. The primary continuously sends WAL records to connected replicas over a replication connection.

**Key settings on the primary:**

- `wal_level = replica` — includes enough information in WAL for replicas to replay
- `max_wal_senders` — maximum concurrent replication connections
- `max_replication_slots` — replication slots reserve WAL so replicas don't fall behind

**Key settings on the replica:**

- `hot_standby = on` — allows read queries while replaying WAL
- `primary_conninfo` — connection string to the primary (Patroni manages this)

### WAL (Write-Ahead Log)

The write-ahead log is PostgreSQL's transaction log. Every data modification is written to WAL **before** it is applied to data files. This ensures durability (crash recovery) and enables replication (replicas replay the same WAL).

WAL segments are 16 MB files in `pg_wal/`. They accumulate until:
- A checkpoint completes (data files synced to disk)
- `max_wal_size` is reached (forces a checkpoint)
- Archiving/cleanup removes old segments

### Replication Slots

A replication slot is a server-side bookmark that tells the primary: *"Do not remove WAL segments until this replica has received them."*

Without slots, a slow or disconnected replica could cause the primary to recycle WAL the replica still needs — breaking replication and requiring a full `pg_basebackup` to recover. **Patroni always uses replication slots** (`use_slots: true`).

### Failover vs. Switchover

| Aspect | Failover | Switchover |
|--------|----------|------------|
| **Trigger** | Unplanned — primary crashes, network partition, OOM | Planned — maintenance, OS upgrades, hardware replacement |
| **Initiation** | Automatic (Patroni detects failure) | Manual (`patronictl switchover`) |
| **Data Loss Risk** | Possible (async replication) | Zero (graceful handoff) |
| **Old Primary** | May still be running (split-brain risk) | Gracefully demoted to replica |
| **Speed** | Seconds to ~40s (depends on TTL) | Near-instant |

### Split Brain

**Split brain** occurs when two nodes believe they are the primary at the same time and both accept writes. This corrupts data irrecoverably.

**Causes:**
- Network partition: primary and replica lose contact, both think the other is down
- etcd quorum loss: multiple Patroni nodes think they hold the leader lock
- Clock drift: etcd election timeouts fire incorrectly

**Protection (all built into this cluster):**
- etcd quorum requirement (a majority of nodes must agree)
- Patroni TTL-based leader lock (expires if Patroni stops heartbeating)
- Fencing (watchdog / STONITH) — forcibly reboots a node that loses quorum
- `maximum_lag_on_failover` — prevents promotion of severely lagged replicas

### Leader Election and Distributed Consensus

Patroni uses etcd's **Raft consensus algorithm** for leader election. Simplified:

1. Each Patroni node tries to acquire a lock in etcd: `/service/<scope>/leader`
2. The lock is a key with a **TTL** (time-to-live, e.g. 30 seconds)
3. The node that creates the key becomes leader; the others become followers
4. The leader must **renew the lock** (heartbeat) before the TTL expires
5. If the leader stops heartbeating (crash, network), the lock expires
6. Followers race to acquire the expired lock — the winner becomes the new leader

### Why etcd? Because It Provides:

- **Strong consistency** — linearizable reads/writes via Raft
- **TTL/leases** — automatic lock expiration
- **Watch mechanism** — Patroni gets instant notification of changes
- **Cluster membership** — dynamic node discovery

### etcd Quorum

etcd uses Raft, which requires a majority (**quorum**) to operate:

| Cluster Size | Quorum Required | Tolerated Failures |
|--------------|-----------------|--------------------|
| 1 | 1 | 0 (no HA) |
| **3** | **2** | **1** |
| 5 | 3 | 2 |
| 7 | 4 | 3 |

This is why we use **3 etcd nodes** — it tolerates exactly one node failure while maintaining quorum. With 2 nodes, quorum = 2, so zero failures are tolerated (which defeats the purpose of HA).

---

## 4. Patroni Internals — How the Brain Works

### How Patroni Starts

When `systemctl start patroni` runs:

1. Patroni reads `/etc/patroni/patroni.yml`
2. Connects to the etcd endpoints listed in `etcd3.hosts`
3. Checks whether a cluster with this **scope** already exists in etcd
4. **If no cluster exists** and this node has `bootstrap.method: initdb`:
   - Runs `initdb` to create a new PostgreSQL data directory
   - Creates users, replication slots, and configuration
   - Acquires the leader lock in etcd → becomes primary
5. **If a cluster exists:**
   - Reads the current leader from etcd
   - If this node was previously primary and has valid data → tries to acquire the leader lock; if successful, promotes PostgreSQL to primary; if another node holds the lock, starts as a replica
   - If this node was previously a replica (or is new): checks whether a data directory exists and is valid; if not, runs `pg_basebackup` from the current primary; starts PostgreSQL as a replica; registers as a follower in etcd

### How the First PostgreSQL Node Becomes Primary (Bootstrap)

1. Patroni starts on db1
2. Reads `patroni.yml` → `scope: "kyc"`, `bootstrap.method: initdb`
3. Connects to etcd → no existing cluster for this scope
4. Patroni runs `initdb`:
   - Creates the data directory with encoding UTF8
   - Enables **data checksums** (detects corruption)
   - Sets auth method to `scram-sha-256`
   - Creates users: `postgres` (superuser), `replicator` (replication), `pgpool` (monitoring), `admin` (management)
5. Patroni writes the initial cluster state to etcd (`/percona_lab/kyc/leader`, `.../members/db1`, `.../config`, `.../history`)
6. Patroni starts PostgreSQL as **PRIMARY**
7. Patroni begins heartbeating the leader lock (renews TTL every `loop_wait` seconds)

### How Replicas Join

1. Patroni starts on db2
2. Connects to etcd → finds existing cluster, leader = db1
3. Checks local data directory → empty or invalid → proceeds to `pg_basebackup`
4. Runs `pg_basebackup`:
   - Connects to db1:5432 as `replicator` user
   - Streams base backup + WAL to the local data directory
   - Creates `standby.signal` (tells PostgreSQL to start as a replica)
5. Patroni starts PostgreSQL as **REPLICA**
6. Patroni registers in etcd: `/percona_lab/kyc/members/db2` → `{role: "replica"}`
7. Patroni begins streaming replication from the primary

### How Patroni Communicates with etcd

| Operation | etcd Key | Purpose |
|-----------|----------|---------|
| Leader lock | `/percona_lab/kyc/leader` | Key with TTL; only the leader can write it |
| Member registration | `/percona_lab/kyc/members/<name>` | Node metadata, role, state, timeline |
| Cluster config | `/percona_lab/kyc/config` | PostgreSQL parameters (dynamic) |
| Timeline history | `/percona_lab/kyc/history` | Used by `pg_rewind` after failover |
| Watches | All keys above | Instant notification of changes |

**Heartbeat loop** (runs every `loop_wait` seconds, default 10s):
- If leader → renew the leader lock TTL
- If follower → check whether the leader lock has expired
- Read cluster state from etcd
- Reconcile local PostgreSQL state with the desired state
- Apply configuration changes from etcd to `postgresql.conf`
- Update local state in etcd

### How Leader Locks Work

In `patroni.yml` `bootstrap.dcs`:

```yaml
ttl: 30           # Lock expires after 30s without renewal
loop_wait: 10     # Patroni heartbeat interval
retry_timeout: 10 # Wait before retrying failed operations
```

**Scenario: Primary crashes at t=0**

| Time | Event |
|------|-------|
| t=0 | db1 (primary) crashes — Patroni stops, no more heartbeats |
| t=10 | db2 Patroni loop runs, sees the leader lock still valid (expires at t=30) |
| t=20 | db2 Patroni loop runs, leader lock still valid |
| t=30 | **Leader lock expires** in etcd (TTL reached) |
| t=30–40 | db2 (and db3) next loop iteration → both try to acquire the lock |
| t=30–40 | One wins (Raft consensus), becomes the new leader |
| t=30–40 | Winner promotes local PostgreSQL to primary |
| t=30–40 | Winner writes the new leader key to etcd |
| t=40 | Other node sees the new leader, becomes a replica |

**Failover time ≈ TTL + loop_wait** (30s + 10s = ~40s worst case with these defaults).

### How Failover Happens — Step by Step

```
NORMAL STATE:
etcd: leader = db1 (TTL=30s, renewed every 10s)
db1:  PostgreSQL PRIMARY, Patroni LEADER
db2:  PostgreSQL REPLICA, Patroni FOLLOWER
db3:  PostgreSQL REPLICA, Patroni FOLLOWER

FAILURE:
1. db1 crashes (power loss, kernel panic, OOM kill)
2. Patroni on db1 stops → no more leader lock renewals
3. The etcd leader lock TTL counts down... expires at t=30s

ELECTION:
4. db2 Patroni loop (t=30-40s): detects the expired lock
5. db2 attempts to write a new leader key with its identity
6. etcd Raft consensus: db2 wins (or db3, but only one)
7. db2 Patroni: "I am leader now"

PROMOTION:
8. db2 Patroni calls pg_ctl promote
9. PostgreSQL on db2 ends recovery and becomes PRIMARY
10. db2 Patroni writes a new leader key to etcd with a new TTL
11. db2 Patroni updates member state in etcd: role=leader

RECONCILIATION:
12. db3 Patroni loop: sees new leader = db2
13. db3 updates local state: role=replica, follows db2
14. db3 ensures the replication connection points to db2

CLIENT ROUTING (pgpool-II):
15. pgpool-II health check detects db1 down, db2 up as primary
16. pgpool-II routes writes to db2
17. The VIP may move independently (pgpool-II watchdog leader election)
```

---

## 5. pgpool-II — The Traffic Controller

### What pgpool-II Does

pgpool-II sits between applications and PostgreSQL. It provides:

| Feature | Description |
|---------|-------------|
| **Connection Pooling** | Reuses PostgreSQL connections — reduces overhead of frequent connect/disconnect |
| **Read/Write Splitting** | Sends SELECT to replicas, INSERT/UPDATE/DELETE to the primary (via `load_balance_mode`) |
| **Primary Detection** | Uses streaming replication checks (`sr_check`) to determine which backend is primary |
| **Virtual IP (VIP) Management** | The **watchdog** module manages a floating IP — applications connect to the VIP, not to individual nodes |
| **Failover Detection** | Health checks (`health_check_period`) detect backend failures |
| **PCP (Pgpool Control Protocol)** | Administrative interface for `pcp_*` commands (attach/detach nodes, promote, etc.) |

### What pgpool-II Does NOT Do

| Not pgpool-II's Job | Handled By |
|---------------------|------------|
| PostgreSQL replication management | Patroni |
| Leader election for PostgreSQL | Patroni + etcd |
| pg_basebackup / pg_rewind | Patroni |
| PostgreSQL configuration management | Patroni (via etcd) |
| Data consistency / split-brain prevention for PostgreSQL | Patroni + etcd |

### Why Patroni Manages PostgreSQL HA, and pgpool-II Sits in Front

**Separation of concerns:**

```
Patroni's Domain (Data Plane):
├── Which PostgreSQL node is primary?
├── Streaming replication setup
├── Failover / switchover execution
├── Replication slot management
├── Configuration distribution
└── Timeline history

pgpool-II's Domain (Access Plane):
├── Where do applications connect? (VIP)
├── Connection pooling
├── Query routing (read/write split)
├── Backend health monitoring
└── Failover signaling to applications (via the VIP)
```

Why this matters: If pgpool-II tried to manage PostgreSQL failover, you would have two systems fighting over who is primary. **Patroni is the single source of truth for PostgreSQL leadership. pgpool-II follows Patroni's lead.**

### The Watchdog and the Virtual IP

The pgpool-II **watchdog** runs on all 3 nodes. The members elect a **watchdog leader** (by priority). The watchdog leader:
- Owns the **floating VIP** (`192.168.122.200`)
- Monitors the other pgpool nodes' heartbeats (UDP 9000)
- If the watchdog leader dies, another member **takes over the VIP** within seconds

This means applications always have ONE address to connect to, and that address **never goes down** — as long as at least 2 of 3 pgpool nodes are alive.

### How Applications Connect

```
Application
    │
    ▼
┌────────────────────────────────────────────┐
│  pgpool-II VIP: 192.168.122.200:9999       │
│  (floats between db1, db2, db3 via watchdog)│
└────────────────────────────────────────────┘
    │
    ├──► Write queries ──► Current PostgreSQL Primary (detected via sr_check)
    │
    └──► Read queries  ──► Load balanced across Replicas (if load_balance_mode=on)
```

```bash
# From psql
psql -h 192.168.122.200 -p 9999 -U postgres -d postgres

# From an application
postgresql://postgres:***@192.168.122.200:9999/postgres
```

---

## 6. Server Planning — Before You Install Anything

### Reference Architecture (3 Nodes + 1 Backup Node)

| Node | IP Address | Components |
|------|-----------|------------|
| db1 (`percona-node-1`) | 192.168.122.150 | PostgreSQL 16, Patroni, etcd, pgpool-II |
| db2 (`percona-node-2`) | 192.168.122.151 | PostgreSQL 16, Patroni, etcd, pgpool-II |
| db3 (`percona-node-3`) | 192.168.122.152 | PostgreSQL 16, Patroni, etcd, pgpool-II |
| db-backup (`percona-pgbackrest`) | 192.168.122.153 | pgBackRest server + PMM Server (Docker) |

**Applications connect to the Virtual IP `192.168.122.200:9999`** (served by whichever node is the pgpool-II watchdog leader).

### Why 3 Database Nodes?

- **PostgreSQL:** 1 primary + 2 replicas (tolerates 1 replica loss and still has HA)
- **etcd:** 3 nodes = quorum of 2 (tolerates 1 etcd node loss)
- **pgpool-II:** 3 nodes = watchdog quorum of 2 (tolerates 1 pgpool node loss)

### Hardware Sizing Guidelines (Per Node)

| Resource | Minimum | Recommended | Notes |
|----------|---------|-------------|-------|
| CPU | 4 vCPU | 8+ vCPU | Patroni + PostgreSQL + etcd + pgpool-II all run here |
| RAM | 8 GB | 32+ GB | PostgreSQL `shared_buffers` = 25–40% of RAM; etcd needs low-latency memory |
| Disk (OS) | 50 GB | 100 GB | Root filesystem |
| Disk (PostgreSQL data) | 100 GB | 500+ GB NVMe | Separate disk for `/postgres/data` — critical for performance |
| Disk (etcd) | 10 GB | 50 GB NVMe | Separate disk for `/var/lib/etcd` — etcd is latency-sensitive |
| Network | 1 Gbps | 10 Gbps | Low latency between nodes is essential for replication and etcd |

### DNS and Hostname Resolution

All nodes must resolve each other by hostname. Configure via `/etc/hosts` or internal DNS:

```
# /etc/hosts on ALL nodes
192.168.122.150  db1
192.168.122.151  db2
192.168.122.152  db3
192.168.122.153  db-backup
192.168.122.200  pgpool-vip
```

**Test:** `ping db1`, `ping db2`, `ping db3` from each node — all must succeed.

---

## 7. What's Inside This Repository

```
patroni-ansible/
├── site.yml                        ← Master playbook: runs 01→06 in order
├── hosts                           ← Inventory: 4 nodes (3 PG + 1 backup)
├── ansible.cfg                     ← Ansible settings (root-based, no sudo prompts)
├── README.md                       ← This file
├── 01_Install_Percona.yml          ← Repos, packages, prerequisites
├── 02_Configure_Etcd.yml           ← 3-node etcd cluster
├── 03_Configure_Patroni.yml        ← PostgreSQL 16 + Patroni HA bootstrap
├── 04_Configure_Pgpool.yml         ← pgpool-II 4.5 + Watchdog + VIP
├── 05_Configure_Pgbackrest.yml     ← pgBackRest server + client integration
└── 06_Install_Pmm_Monitoring.yml   ← PMM Server (Docker) + PMM Client
```

> 💡 **Design choice:** every config file is written **inline** in the playbooks (via `copy: content: |`) — no `templates/` directory. This makes each playbook fully self-contained: you see the exact config being deployed without opening another file.

### Playbook Sequence

| # | Playbook | What It Does (Plain English) |
|---|----------|------------------------------|
| 01 | `01_Install_Percona.yml` | Adds the Percona repository, enables EPEL + CRB, installs PostgreSQL 16, Patroni, etcd, pgpool-II, pgBackRest, jq — and **purges any old/broken installs** so you start clean |
| 02 | `02_Configure_Etcd.yml` | Writes the etcd config on all 3 nodes, **wipes stale etcd data** (so a re-run bootstraps cleanly), starts etcd, and verifies quorum |
| 03 | `03_Configure_Patroni.yml` | Creates the PostgreSQL data directory, writes `patroni.yml` (the full HA config), installs the systemd unit, **starts the primary first**, waits, then starts replicas — then verifies with `patronictl list` |
| 04 | `04_Configure_Pgpool.yml` | Writes `pgpool.conf` + `pgpool_watchdog.conf` + `pool_hba.conf` + `pool_passwd` + `pcp.conf`, deploys Patroni-aware `failover.sh` / `follow_master.sh`, sets `pgpool_node_id`, and starts the watchdog cluster so the VIP is claimed |
| 05 | `05_Configure_Pgbackrest.yml` | Installs/connects pgBackRest on the backup node, exchanges SSH keys with all PG nodes, writes `pgbackrest.conf` with stanza `kyc`, and prints the exact commands to create the stanza + first backup |
| 06 | `06_Install_Pmm_Monitoring.yml` | Pulls and runs the PMM Server Docker container on the backup node, opens the firewall for it, installs PMM Client + `pg_stat_monitor` on all 3 PG nodes, and registers them with the server |

### Default Values You Should Know

| Setting | Value |
|---------|-------|
| Cluster scope (PostgreSQL name) | `kyc` |
| Patroni namespace | `percona_lab` |
| PostgreSQL version | 16 |
| Data directory | `/postgres/data/16/kyc` |
| etcd token | `PostgreSQL_HA_Cluster_1` |
| etcd data directory | `/var/lib/etcd` |
| **Floating VIP** | **`192.168.122.200`** on port **9999** |
| VIP network interface | `eth0` (change if your NIC differs) |
| pgBackRest stanza | `kyc`, repo at `/postgres/pgbackup` |
| PMM Server URL | `https://192.168.122.153:443` |
| Patroni REST API | `:8008` on every node |
| PCP port / user | `9898` / `pgpool_pcp` |

---

## 8. Deployment Method A — Ansible (Automated)

This is the recommended path: one command deploys the whole cluster, and every playbook is **idempotent** — you can re-run it safely.

### Step 1 — Prerequisites

- **4 CentOS Stream 9 (or RHEL 9) VMs** with:
  - root SSH access (or a sudo user)
  - Outbound internet access (to reach Percona / EPEL / Docker repos)
- **Ansible 2.16+** installed on the machine you run from (your laptop or the VPS):
  ```bash
  # On CentOS/RHEL
  sudo dnf install -y ansible-core
  # On Ubuntu/Debian
  sudo apt install -y ansible
  # Or via pip in a venv
  python3 -m venv ~/ansible-venv && source ~/ansible-venv/bin/activate
  pip install ansible
  ```
- **Network connectivity** between all nodes (ports 2379, 2380, 5432, 8008, 9000, 9898, 9999)
- **VIP `192.168.122.200`** must be unused and routable on the same subnet
- The correct NIC name on your VMs (this repo uses `eth0`; check with `ip link`)

### Step 2 — Clone and Edit the Inventory

```bash
git clone https://github.com/marufmoinuddin/patroni-ansible.git
cd patroni-ansible
vim hosts
```

The inventory looks like this:

```ini
[pg_nodes]
percona-node-1 ansible_host=192.168.122.150 ansible_user=root
percona-node-2 ansible_host=192.168.122.151 ansible_user=root
percona-node-3 ansible_host=192.168.122.152 ansible_user=root

[pg_backrest]
percona-pgbackrest ansible_host=192.168.122.153 ansible_user=root
```

Replace the IPs with your own. The `ansible_user=root` means Ansible connects as root directly — make sure your SSH key is authorized on every node:

```bash
ssh-copy-id root@192.168.122.150
ssh-copy-id root@192.168.122.151
ssh-copy-id root@192.168.122.152
ssh-copy-id root@192.168.122.153
```

### Step 3 — Adjust Variables (Important!)

All secrets are now managed in a separate **`credentials.yaml`** file (not committed to git). This allows you to encrypt it with `ansible-vault` and keep passwords out of playbooks.

**Quick start:**
```bash
# 1. Copy the example and edit with your real values
cp credentials.yaml.example credentials.yaml
vim credentials.yaml

# 2. (Recommended) Encrypt it with ansible-vault
ansible-vault encrypt credentials.yaml

# 3. Edit later with: ansible-vault edit credentials.yaml
```

**These are the values you most likely need to change in `credentials.yaml`:**

| Variable | Description | Example |
|----------|-------------|---------|
| `patroni_scope` | Cluster name | `kyc` |
| `postgres_password` | PostgreSQL superuser password | **strong random** |
| `replicator_password` | Replication user password | **strong random** |
| `patroni_admin_password` | Patroni REST API admin | **strong random** |
| `percona_password` | Percona monitoring user | **strong random** |
| `pgpool_password` | pgpool monitoring user | **strong random** |
| `pcp_password` | Pgpool PCP admin user | **strong random** |
| `pmm_admin_password` | PMM web UI admin | **strong random** |
| `pg_pmm_user_password` | PMM PostgreSQL monitor user | **strong random** |
| `vip_address` | Floating Virtual IP | `192.168.122.200` |
| `vip_interface` | NIC for VIP (check `ip link`) | `eth0` |

> ⚠️ **Never commit `credentials.yaml` to git** — it's in `.gitignore`. Only commit `credentials.yaml.example`.

### Step 4 — Run the Deployment

```bash
# Check the whole suite parses correctly
ansible-playbook -i hosts site.yml --syntax-check

# Run everything (this takes ~10-15 minutes)
# If you encrypted credentials.yaml with ansible-vault:
ansible-playbook -i hosts site.yml --ask-vault-pass

# If you did NOT encrypt (not recommended for production):
ansible-playbook -i hosts site.yml
```

> ⏳ What you'll see: play 01 installs packages on all nodes (slowest), play 02 forms the etcd quorum, play 03 bootstraps PostgreSQL with Patroni (primary first, then replicas), play 04 starts the pgpool watchdog cluster, play 05 wires up pgBackRest, play 06 brings up PMM. **Green = done.**

### Step 5 — Post-Deployment Checklist

1. **Patroni cluster is green:**
   ```bash
   patronictl -c /etc/patroni/patroni.yml list
   ```
   You should see one `Leader` and two `Streaming` replicas with `0` lag.

2. **etcd quorum is healthy:**
   ```bash
   ETCDCTL_API=3 etcdctl --endpoints=http://192.168.122.150:2379 endpoint health
   ETCDCTL_API=3 etcdctl --endpoints=http://192.168.122.150:2379 member list
   ```

3. **pgpool watchdog elected a leader and owns the VIP:**
   ```bash
   pcp_watchdog_info -h localhost -p 9898 -U pgpool_pcp -w
   ip addr show eth0 | grep 192.168.122.200
   ```

4. **You can connect through the VIP:**
   ```bash
   psql -h 192.168.122.200 -p 9999 -U postgres -d postgres -c "SELECT 1;"
   ```

5. **Create the pgBackRest stanza + first backup** (the playbook prints these, by design — they're intentionally manual so *you* decide when to take the first backup):
   ```bash
   sudo -iu postgres pgbackrest --stanza=kyc stanza-create
   sudo -iu postgres pgbackrest --stanza=kyc --type=full backup
   sudo -iu postgres pgbackrest --stanza=kyc info
   ```

6. **Log into PMM:**
   - URL: `https://192.168.122.153:443`
   - User: `admin` / your chosen password
   - You should see 3 PostgreSQL nodes reporting metrics

### Rerunning Safely

Every playbook is idempotent. If something failed midway, fix it and re-run:

```bash
ansible-playbook -i hosts site.yml
```

Only play 02 wipes etcd data **by design** (it bootstraps a fresh cluster). If your cluster is already healthy and you re-run, Patroni will simply reconnect — **your data is safe**.

---

## 9. Deployment Method B — Manual (No Ansible)

Prefer to see every screw and bolt? This section walks through exactly what the playbooks automate, hand-by-hand. Run these on **all nodes** unless stated otherwise.

**Pick your distro family** — commands differ only in package management and a few paths. The logic (etcd → Patroni → pgpool → pgBackRest → PMM) is identical.

| Area | RHEL / CentOS / Stream 9 | Debian / Ubuntu (22.04, 24.04) |
|------|-------------------------|--------------------------------|
| Package manager | `dnf` | `apt` (with `apt update`) |
| Percona repo | RPM + `percona-release setup ppg-16` | `.deb` + `percona-release setup ppg-16` |
| PostgreSQL data dir | `/var/lib/pgsql/16/data/kyc` | `/postgres/data/16/kyc` |
| PostgreSQL bin dir | `/usr/pgsql-16/bin` | `/usr/lib/postgresql/16/bin` |
| PostgreSQL service | `postgresql-16` (systemd) | `postgresql` (via `pg_ctlcluster`) |
| Patroni binary | `/usr/bin/patroni` | `/bin/patroni` |
| Pgpool config dir | `/etc/pgpool-II` | `/etc/pgpool2` |
| Pgpool service name | `pgpool` | `pgpool2` |
| Postgres user home | `/var/lib/pgsql` | `/var/lib/postgresql` |

---

### Phase 0 — OS Preparation (all 4 nodes)

```bash
# 1. Hostnames
hostnamectl set-hostname db1      # db2, db3, db-backup respectively

# 2. /etc/hosts — same on every node
cat >> /etc/hosts <<'EOF'
192.168.122.150  db1
192.168.122.151  db2
192.168.122.152  db3
192.168.122.153  db-backup
192.168.122.200  pgpool-vip
EOF

# 3. Firewall: open cluster ports (between nodes)
# RHEL/CentOS:
firewall-cmd --permanent --add-port={2379,2380}/tcp
firewall-cmd --permanent --add-port=5432/tcp
firewall-cmd --permanent --add-port=8008/tcp
firewall-cmd --permanent --add-port=9000/tcp
firewall-cmd --permanent --add-port={9898,9999}/tcp
firewall-cmd --reload

# Debian/Ubuntu (UFW):
ufw allow 2379/tcp
ufw allow 2380/tcp
ufw allow 5432/tcp
ufw allow 8008/tcp
ufw allow 9000/tcp
ufw allow 9898/tcp
ufw allow 9999/tcp
ufw --force enable
```

---

### Phase 1 — Install Percona Packages (all 3 DB nodes + backup node)

#### RHEL / CentOS / Stream 9

```bash
# 1. Enable EPEL and CRB (CodeReady Builder) — needed for libssh2 etc.
dnf install -y epel-release
dnf config-manager --set-enabled crb

# 2. Install Percona release RPM (GPG key comes with the package)
dnf install -y https://repo.percona.com/yum/percona-release-latest.noarch.rpm

# 3. Enable the PostgreSQL 16 Percona repository
percona-release setup ppg-16

# 4. Install everything
dnf install -y \
  percona-postgresql16-server \
  percona-patroni percona-patroni-etcd etcd jq \
  percona-pgpool-II-pg16 percona-pgpool-II-pg16-extensions \
  percona-pgbackrest
```

#### Debian / Ubuntu

```bash
# 1. Install prerequisites
apt update && apt install -y curl wget gnupg2

# 2. Download and install Percona release .deb
wget https://repo.percona.com/apt/percona-release_latest.generic_all.deb
apt install -y ./percona-release_latest.generic_all.deb

# 3. Enable the PostgreSQL 16 Percona repository
percona-release setup ppg-16

# 4. Install everything
apt update && apt install -y \
  percona-postgresql-16 \
  percona-patroni etcd \
  percona-pgpool16 postgresql-16-pgpool2 \
  percona-pgbackrest
```

---

### Phase 2 — etcd Cluster (db1, db2, db3)

Create `/etc/etcd/etcd.conf` — the token and member IPs **must be identical** on all three nodes; only `name` and `initial-advertise-peer-urls` differ:

```ini
# /etc/etcd/etcd.conf  (db1 example)
ETCD_NAME="db1"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.122.150:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.122.150:2380"
ETCD_INITIAL_CLUSTER="db1=http://192.168.122.150:2380,db2=http://192.168.122.151:2380,db3=http://192.168.122.152:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="PostgreSQL_HA_Cluster_1"
```

> Swap `ETCD_NAME` and the two `...ADVERTISE...` values on db2 (`192.168.122.151`) and db3 (`192.168.122.152`).

Create the systemd unit (identical on both distros):

```bash
cat > /etc/systemd/system/etcd.service <<'EOF'
[Unit]
Description=etcd key-value store
Documentation=https://etcd.io/docs/
After=network.target

[Service]
Environment="TOKEN=PostgreSQL_HA_Cluster_1"
Environment="CLUSTER_STATE=new"
Environment="THIS_NAME={{ ansible_hostname }}"
Environment="THIS_IP={{ ansible_default_ipv4.address }}"
Environment="CLUSTER=db1=http://192.168.122.150:2380,db2=http://192.168.122.151:2380,db3=http://192.168.122.152:2380"

ExecStart=/usr/bin/etcd \
  --data-dir=/var/lib/etcd \
  --name ${THIS_NAME} \
  --initial-advertise-peer-urls http://${THIS_IP}:2380 \
  --listen-peer-urls http://${THIS_IP}:2380 \
  --advertise-client-urls http://${THIS_IP}:2379 \
  --listen-client-urls http://${THIS_IP}:2379 \
  --initial-cluster ${CLUSTER} \
  --initial-cluster-state ${CLUSTER_STATE} \
  --initial-cluster-token ${TOKEN}

Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

```bash
# Start etcd on ALL THREE nodes (roughly together — quorum needs 2/3)
systemctl daemon-reload
systemctl enable --now etcd

# Verify quorum from any node
ETCDCTL_API=3 etcdctl --endpoints=http://192.168.122.150:2379 endpoint health
ETCDCTL_API=3 etcdctl --endpoints=http://192.168.122.150:2379 member list
# All 3 members must show "healthy: true"
```

---

### Phase 3 — Patroni + PostgreSQL (db1, db2, db3)

Create `/etc/patroni/patroni.yml` on **each** node. The structure is the same everywhere; only `name` and the two `connect_address`/etcd host lines change per node.

```yaml
# /etc/patroni/patroni.yml  (db1 example)
namespace: percona_lab
scope: kyc
name: db1                       # db2 / db3 on the other nodes

restapi:
  listen: 0.0.0.0:8008
  connect_address: 192.168.122.150:8008   # this node's IP

etcd3:
  host: 192.168.122.150:2379               # this node's etcd

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576

    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        wal_level: replica
        hot_standby: "on"
        max_wal_senders: 5
        max_replication_slots: 10
        wal_log_hints: "on"
        logging_collector: "on"
        max_wal_size: '10GB'
        archive_mode: "on"
        archive_timeout: 600s
        archive_command: "cp -f %p /postgres/pgbackup/kyc/archive/%f"

  initdb:
    - encoding: UTF8
    - data-checksums

  pg_hba:
    - host replication replicator 127.0.0.1/32 trust
    - host replication replicator 0.0.0.0/0 scram-sha-256
    - host all all 0.0.0.0/0 scram-sha-256
    - host all all ::0/0 scram-sha-256

  users:
    admin:
      password: CHANGE_ME_ADMIN
      options:
        - createrole
        - createdb
    percona:
      password: CHANGE_ME_PERCONA
      options:
        - createrole
        - createdb
    pgpool:
      password: CHANGE_ME_PGPOOL
      options:
        - createrole
        - createdb

postgresql:
  cluster_name: cluster_1
  listen: 0.0.0.0:5432
  connect_address: 192.168.122.150:5432   # this node's IP
  data_dir: /postgres/data/16/kyc         # DEBIAN PATH — change for RHEL below
  bin_dir: /usr/lib/postgresql/16/bin      # DEBIAN PATH — change for RHEL below
  pgpass: /tmp/pgpass0
  authentication:
    replication:
      username: replicator
      password: CHANGE_ME_REPLICATOR
    superuser:
      username: postgres
      password: CHANGE_ME_POSTGRES
  parameters:
    unix_socket_directories: /var/run/postgresql
  create_replica_methods:
    - basebackup
  basebackup:
    checkpoint: 'fast'

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false
```

> 📋 **Path differences for `patroni.yml`:**
> - **RHEL/CentOS:** `data_dir: /var/lib/pgsql/16/data/kyc`, `bin_dir: /usr/pgsql-16/bin`
> - **Debian/Ubuntu:** `data_dir: /postgres/data/16/kyc`, `bin_dir: /usr/lib/postgresql/16/bin`

#### Initialize PostgreSQL + systemd unit

##### RHEL / CentOS / Stream 9

```bash
# Data directory will be created by Patroni on bootstrap
mkdir -p /var/lib/pgsql/16 /etc/patroni
chown -R postgres:postgres /var/lib/pgsql

# systemd unit (Patroni binary at /usr/bin/patroni)
cat > /etc/systemd/system/patroni.service <<'EOF'
[Unit]
Description=Runners to orchestrate a high-availability PostgreSQL
After=syslog.target network.target etcd.service

[Service]
Type=simple
User=postgres
Group=postgres
ExecStart=/usr/bin/patroni /etc/patroni/patroni.yml
ExecReload=/bin/kill -s HUP $MAINPID
KillMode=process
TimeoutSec=30
Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
```

##### Debian / Ubuntu

```bash
# Create the cluster with pg_createcluster (on PRIMARY ONLY - db1)
mkdir -p /postgres/data/16 /etc/patroni
chown -R postgres:postgres /postgres

# On PRIMARY (db1) only:
pg_createcluster 16 kyc -d /postgres/data/16/kyc

# Stop it so Patroni can take over
pg_ctlcluster 16 kyc stop

# systemd unit (Patroni binary at /bin/patroni)
cat > /etc/systemd/system/patroni.service <<'EOF'
[Unit]
Description=Runners to orchestrate a high-availability PostgreSQL
After=syslog.target network.target etcd.service

[Service]
Type=simple
User=postgres
Group=postgres
ExecStart=/bin/patroni /etc/patroni/patroni.yml
ExecReload=/bin/kill -s HUP $MAINPID
KillMode=process
TimeoutSec=30
Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
```

#### Bootstrap the cluster — start the primary FIRST

```bash
# On db1 ONLY — this creates the data directory and makes db1 the leader
systemctl enable --now patroni

# Wait ~30 seconds for initdb + leader lock
patronictl -c /etc/patroni/patroni.yml list
# db1 should appear as Leader

# Now start replicas — db2, db3
systemctl enable --now patroni

# Wait ~1-2 minutes for pg_basebackup + catch-up
patronictl -c /etc/patroni/patroni.yml list
# db1 Leader, db2 Streaming, db3 Streaming, lag 0
```

> 🧠 **Why primary first?** If you start a replica before any primary exists, Patroni would *also* try to bootstrap — two nodes racing to initdb is chaos. The leader lock in etcd prevents a real split-brain, but starting in order is clean and predictable.

---

### Phase 4 — pgpool-II + Watchdog + VIP (db1, db2, db3)

The full `pgpool.conf` is long — here is the **watchdog-relevant essence** (per node; the `pgpool_node_id` file differs: `0`, `1`, `2`).

> 📋 **Config directory:**
> - **RHEL/CentOS:** `/etc/pgpool-II`
> - **Debian/Ubuntu:** `/etc/pgpool2`

```ini
# pgpool.conf  (db1 example — key lines)
listen_addresses = '*'
port = 9999
socket_dir = '/var/run/pgpool'

backend_hostname0 = '192.168.122.150'
backend_port0 = 5432
backend_weight0 = 1
backend_data_directory0 = '/postgres/data/16/kyc'   # DEBIAN PATH — change for RHEL
backend_flag0 = 'ALLOW_TO_FAILOVER'

backend_hostname1 = '192.168.122.151'
backend_port1 = 5432
backend_weight1 = 1
backend_data_directory1 = '/postgres/data/16/kyc'   # DEBIAN PATH — change for RHEL
backend_flag1 = 'ALLOW_TO_FAILOVER'

backend_hostname2 = '192.168.122.152'
backend_port2 = 5432
backend_weight2 = 1
backend_data_directory2 = '/postgres/data/16/kyc'   # DEBIAN PATH — change for RHEL
backend_flag2 = 'ALLOW_TO_FAILOVER'

# Health checks
health_check_period = 10
health_check_timeout = 20
health_check_user = 'pgpool'
health_check_password = 'CHANGE_ME_HEALTH'

# Streaming replication check
sr_check_period = 10
sr_check_user = 'pgpool'
sr_check_password = 'CHANGE_ME_HEALTH'
delay_threshold = 1048576

# PCP
pcp_listen_addresses = '*'
pcp_port = 9898
pcp_socket_dir = '/var/run/pgpool'
```

> 📋 **Backend data_directory paths:**
> - **RHEL/CentOS:** `/var/lib/pgsql/16/data/kyc`
> - **Debian/Ubuntu:** `/postgres/data/16/kyc`

And the watchdog section (must be **indexed** — `wd_port0/1/2` etc. — Pgpool 4.5+ rejects unindexed `wd_*` keys):

```ini
# pgpool_watchdog.conf (db1 example)
use_watchdog = on

wd_hostname = '192.168.122.150'
wd_port = 9000

# Watchdog peers — IMPORTANT: list ALL nodes on every node
wd_hostname0 = '192.168.122.150'
wd_port0 = 9000
wd_hostname1 = '192.168.122.151'
wd_port1 = 9000
wd_hostname2 = '192.168.122.152'
wd_port2 = 9000

# Virtual IP — only active on the watchdog leader
vip = 1
vip_ip = '192.168.122.200'
vip_ifconfig = 'ifconfig eth0:0 192.168.122.200/24'
vip_arping = 'arping -U -I eth0 -c 3 192.168.122.200'
vip_cidr_prefix_length = 24
vip_ip_up_cmd = '/etc/pgpool-II/vip-up.sh'
vip_ip_down_cmd = '/etc/pgpool-II/vip-down.sh'
```

> 📋 **Watchdog config dir and VIP script paths:**
> - **RHEL/CentOS:** `/etc/pgpool-II/`
> - **Debian/Ubuntu:** `/etc/pgpool2/`

Per-node files:

```bash
# db1 → 0, db2 → 1, db3 → 2
echo -n "0" > /etc/pgpool-II/pgpool_node_id    # RHEL path
# echo -n "0" > /etc/pgpool2/pgpool_node_id    # Debian path

# pgpool needs to raise the VIP → sudoers entry
cat > /etc/sudoers.d/pgpool-vip <<'EOF'
pgpool ALL=(root) NOPASSWD: /sbin/ip, /usr/sbin/arping
EOF
chmod 440 /etc/sudoers.d/pgpool-vip
```

Create the PCP passfile and pool password file:

```bash
# PCP user (pgpool_pcp / your password) — generates a pg_md5 hash
pg_md5 -p -u pgpool_pcp   # enter the password when prompted → /etc/pgpool-II/pcp.conf

# PostgreSQL user passwords for pgpool (plaintext file works with SCRAM)
echo "postgres:CHANGE_ME_POSTGRES" > /etc/pgpool-II/pool_passwd
echo "pgpool:CHANGE_ME_PGPOOL"   >> /etc/pgpool-II/pool_passwd
chown pgpool:pgpool /etc/pgpool-II/pool_passwd /etc/pgpool-II/pcp.conf
chmod 600 /etc/pgpool-II/pool_passwd /etc/pgpool-II/pcp.conf
```

> 📋 **Config dir for these files:**
> - **RHEL/CentOS:** `/etc/pgpool-II/`
> - **Debian/Ubuntu:** `/etc/pgpool2/`

Deploy the Patroni-aware failover scripts (simplified):

```bash
# /etc/pgpool-II/failover.sh — on backend failover, query Patroni for the new primary
cat > /etc/pgpool-II/failover.sh <<'EOF'
#!/bin/bash
# pgpool calls this when a backend goes down
# ... query each Patroni REST API :8008 for the leader ...
# ... update pgpool backend list to point at the new primary ...
EOF

# /etc/pgpool-II/follow_master.sh — on primary change, tell replicas to re-follow
cat > /etc/pgpool-II/follow_master.sh <<'EOF'
#!/bin/bash
# Patroni already handles re-following; this script logs the event
EOF

chmod +x /etc/pgpool-II/failover.sh /etc/pgpool-II/follow_master.sh
chown pgpool:pgpool /etc/pgpool-II/failover.sh /etc/pgpool-II/follow_master.sh
```

> 📄 The repository's `04_Configure_Pgpool.yml` contains the **complete, working** versions of every file above, inline. For a manual deployment, copy them from there — they are battle-tested.

Start pgpool on **all three nodes**:

```bash
# RHEL/CentOS:
systemctl enable --now pgpool

# Debian/Ubuntu:
systemctl enable --now pgpool2

# Verify watchdog + VIP from any node
pcp_watchdog_info -h localhost -p 9898 -U pgpool_pcp -w
# One node is "LEADER" and owns 192.168.122.200
ip addr show eth0 | grep 192.168.122.200
```

---

### Phase 5 — pgBackRest (backup node `.153` + PG nodes)

> 📋 **Postgres user home:**
> - **RHEL/CentOS:** `/var/lib/pgsql`
> - **Debian/Ubuntu:** `/var/lib/postgresql`

```bash
# On the backup node (.153)
mkdir -p /postgres/pgbackup
chown -R postgres:postgres /postgres

# SSH key exchange: backup node ↔ each PG node (as postgres user)
# 1. On backup node:  sudo -iu postgres ssh-keygen -t ed25519
# 2. Copy key to each PG node:  sudo -iu postgres ssh-copy-id postgres@db1 (etc.)
# 3. Also allow postgres@dbX to SSH into the backup node (for archiving)

# /etc/pgbackrest.conf on the BACKUP node
cat > /etc/pgbackrest.conf <<'EOF'
[global]
repo1-path = /postgres/pgbackup
repo1-retention-full = 2

[kyc]
pg1-host = 192.168.122.150
pg1-path = /postgres/data/16/kyc      # DEBIAN PATH — change for RHEL
pg1-port = 5432
EOF

# /etc/pgbackrest.conf on each PG NODE (client only, for archiving)
cat > /etc/pgbackrest.conf <<'EOF'
[global]
repo1-host = 192.168.122.153
repo1-path = /postgres/pgbackup

[kyc]
pg1-path = /postgres/data/16/kyc      # DEBIAN PATH — change for RHEL
EOF
```

> 📋 **pgBackRest `pg1-path` values:**
> - **RHEL/CentOS:** `/var/lib/pgsql/16/data/kyc`
> - **Debian/Ubuntu:** `/postgres/data/16/kyc`

```bash
# Create the stanza, then the first full backup (on the backup node)
sudo -iu postgres pgbackrest --stanza=kyc stanza-create
sudo -iu postgres pgbackrest --stanza=kyc --type=full backup
sudo -iu postgres pgbackrest --stanza=kyc info
```

> The `archive_command` in Patroni (Phase 3) sends WAL to `/postgres/pgbackup/kyc/archive/` — pgBackRest picks it up from there. Archiving must be enabled **before** the first backup for a complete PITR chain.

---

### Phase 6 — PMM (backup node `.153` + PG nodes)

#### RHEL / CentOS / Stream 9

```bash
# 1. On the backup node — run PMM Server as a Docker container
dnf install -y docker-ce docker-ce-cli
systemctl enable --now docker

# NOTE: the image listens on 8443 internally; map host 443 → container 8443
docker run -d --name pmm-server --restart always \
  -p 443:8443 -v pmm-data:/srv perconalab/pmm-server:3

# Wait 2-3 minutes, then change the default admin password
docker exec pmm-server change-admin-password YourNewPassword

# 2. On each PG node — install PMM Client
percona-release enable pmm3-client release
dnf install -y pmm3-client percona-pg_stat_monitor16

# 3. Register each node with the server (retry until it succeeds)
pmm-admin config --server-insecure-tls \
  --server-url=https://admin:YourNewPassword@192.168.122.153:443 --force

# 4. Add PostgreSQL monitoring (repeat on each node)
pmm-admin add postgresql --username=pg_pmm --password=CHANGE_ME \
  --service-name=db1-pg --host=localhost --port=5432

# 5. Browse to https://192.168.122.153:443 and watch the dashboards!
```

#### Debian / Ubuntu

```bash
# 1. On the backup node — run PMM Server as a Docker container
apt update && apt install -y docker.io
systemctl enable --now docker

# NOTE: the image listens on 8443 internally; map host 443 → container 8443
docker run -d --name pmm-server --restart always \
  -p 443:8443 -v pmm-data:/srv perconalab/pmm-server:3

# Wait 2-3 minutes, then change the default admin password
docker exec pmm-server change-admin-password YourNewPassword

# 2. On each PG node — install PMM Client
percona-release enable pmm3-client release
apt update && apt install -y pmm3-client percona-pg-stat-monitor16

# 3. Register each node with the server (retry until it succeeds)
pmm-admin config --server-insecure-tls \
  --server-url=https://admin:YourNewPassword@192.168.122.153:443 --force

# 4. Add PostgreSQL monitoring (repeat on each node)
pmm-admin add postgresql --username=pg_pmm --password=CHANGE_ME \
  --service-name=db1-pg --host=localhost --port=5432

# 5. Browse to https://192.168.122.153:443 and watch the dashboards!
```

> 🔥 If the PG nodes cannot reach `https://192.168.122.153:443`, open the firewall on the backup node:
> - **RHEL:** `firewall-cmd --permanent --add-port=443/tcp && firewall-cmd --reload`
> - **Debian:** `ufw allow 443/tcp`
> and add an iptables FORWARD rule for Docker if needed.

---

## 10. Operations Guide — Daily Commands

### Patroni (run on any node)

```bash
# Cluster status — THE command you will use daily
patronictl -c /etc/patroni/patroni.yml list

# Details about one member
patronictl -c /etc/patroni/patroni.yml show-config

# Planned switchover (move primary to db2) — safe, zero downtime
patronictl -c /etc/patroni/patroni.yml switchover

# Restart a node's PostgreSQL (rolling, Patroni-aware)
patronictl -c /etc/patroni/patroni.yml restart kyc

# Failover NOW (promote a specific replica)
patronictl -c /etc/patroni/patroni.yml failover

# Pause automatic failover (maintenance window)
patronictl -c /etc/patroni/patroni.yml pause
# ... work on the cluster ...
patronictl -c /etc/patroni/patroni.yml resume

# History of leaders/timelines
patronictl -c /etc/patroni/patroni.yml history
```

### etcd

```bash
# Health of the quorum
ETCDCTL_API=3 etcdctl --endpoints=http://192.168.122.150:2379 endpoint health

# Member list
ETCDCTL_API=3 etcdctl --endpoints=http://192.168.122.150:2379 member list

# Inspect Patroni keys
ETCDCTL_API=3 etcdctl --endpoints=http://192.168.122.150:2379 \
  get /percona_lab/kyc --prefix

# Who is the current leader?
ETCDCTL_API=3 etcdctl --endpoints=http://192.168.122.150:2379 \
  get /percona_lab/kyc/leader
```

### pgpool-II

```bash
# Watchdog cluster status (who is leader, who owns the VIP)
pcp_watchdog_info -h localhost -p 9898 -U pgpool_pcp -w

# Node status (backend list)
pcp_node_info -h localhost -p 9898 -U pgpool_pcp -w

# Pool status per database
pcp_pool_status -h localhost -p 9898 -U pgpool_pcp -w

# Attach / detach a backend manually
pcp_attach_node -h localhost -p 9898 -U pgpool_pcp -w -n 1
pcp_detach_node -h localhost -p 9898 -U pgpool_pcp -w -n 1

# pgpool logs
journalctl -u pgpool -f
```

### Backups (pgBackRest — backup node)

```bash
# Backup info
sudo -iu postgres pgbackrest --stanza=kyc info

# Full backup
sudo -iu postgres pgbackrest --stanza=kyc --type=full backup

# Incremental backup
sudo -iu postgres pgbackrest --stanza=kyc --type=incr backup

# Restore to a point in time (example)
sudo -iu postgres pgbackrest --stanza=kyc --type=time \
  --target="2026-08-07 12:00:00" restore
```

### Monitoring (PMM)

- **Web UI:** `https://192.168.122.153:443` — PostgreSQL Overview, replication graphs, query analytics
- **CLI from any PG node:**
  ```bash
  pmm-admin list          # what's being monitored
  pmm-admin status        # agent health
  pmm-admin add postgresql   # add a PostgreSQL service
  ```

---

## 11. Failure Testing — Learning by Breaking Things

> 🧪 Do this in a test environment first. It is the only way to trust your HA cluster.

### Test 1 — Kill the Primary (Automatic Failover)

```bash
# 1. From any node, note the current leader
patronictl -c /etc/patroni/patroni.yml list

# 2. SSH to the leader and kill PostgreSQL HARD (simulates a crash)
#    (on the leader node)
systemctl kill -s SIGKILL patroni
#    or even better — power it off:  poweroff

# 3. Wait ~40 seconds (TTL 30s + loop_wait 10s)
# 4. Check the cluster — a replica should now be Leader
patronictl -c /etc/patroni/patroni.yml list

# 5. Check pgpool followed
pcp_watchdog_info -h localhost -p 9898 -U pgpool_pcp -w

# 6. Reconnect the app — it should still work via the VIP with no code change
psql -h 192.168.122.200 -p 9999 -U postgres -d postgres -c "SELECT now();"
```

**Expected result:** writes fail for ~40 seconds, then resume on the new primary. The VIP keeps serving. **No manual intervention.**

### Test 2 — Kill a Replica

```bash
# Stop Patroni on a replica (simulate a node failing)
systemctl stop patroni        # on db3, for example

# The cluster should stay green — Patroni marks db3 as unavailable
patronictl -c /etc/patroni/patroni.yml list

# Restart it — Patroni brings it back as a streaming replica automatically
systemctl start patroni       # on db3
patronictl -c /etc/patroni/patroni.yml list
```

### Test 3 — Kill the pgpool Watchdog Leader

```bash
# 1. Find which pgpool node owns the VIP
pcp_watchdog_info -h localhost -p 9898 -U pgpool_pcp -w

# 2. Stop pgpool on that node
systemctl stop pgpool         # on the watchdog leader

# 3. Within seconds, another node takes over the VIP
ip addr show eth0 | grep 192.168.122.200   # run on other nodes

# 4. Applications keep connecting to the SAME VIP — nothing changed for them
```

### Test 4 — Planned Switchover (Zero Downtime)

```bash
# Move leadership from db1 to db2, gracefully
patronictl -c /etc/patroni/patroni.yml switchover

# Patroni: demotes db1 → promotes db2 → db1 becomes a replica
patronictl -c /etc/patroni/patroni.yml list
```

---

## 12. Troubleshooting Common Issues

| Issue | Likely Cause | Check / Fix |
|-------|--------------|-------------|
| **Patroni won't start** | etcd not reachable | `systemctl status etcd`, `ETCDCTL_API=3 etcdctl endpoint health`; check `patroni.yml` `etcd3.hosts` |
| **etcd quorum not forming** | Stale data from a previous run | `systemctl stop etcd && rm -rf /var/lib/etcd/* && systemctl start etcd` (fresh bootstrap only!) |
| **etcd member fails GPG validation** | Fresh OS missing Percona keys | The RPM installs its own key; the playbook uses `disable_gpg_check: true` for the release RPM |
| **Replicas stuck with lag** | Replication slot missing / WAL removed | `patronictl -c /etc/patroni/patroni.yml list`; check `pg_replication_slots`; a full `pg_basebackup` may be needed |
| **VIP not moving** | sudoers / capability issues | `sudoers.d/pgpool-vip` entry present? `journalctl -u pgpool` for vip_up/down errors |
| **Watchdog not forming** | Firewall 9000, auth key mismatch | Open UDP/TCP 9000 between nodes; `wd_authkey` identical everywhere; nodes reachable |
| **pgpool rejects config** | Unindexed `wd_*` params | Pgpool 4.5+ requires `wd_port0/1/2` (indexed); remove bare `wd_port`, `wd_authkey`, etc. |
| **Cannot connect via VIP** | VIP on wrong node / pgpool not started | `ip addr` (who owns .200?), `pcp_watchdog_info`, `systemctl status pgpool` |
| **pool_passwd auth fails** | MD5 vs SCRAM mismatch | This repo uses a plaintext `pool_passwd` + `pool_hba.conf` (SCRAM-safe); keep file perms 600 |
| **pgBackRest fails** | SSH keys / stanza missing | Run `stanza-create` first; `sudo -iu postgres pgbackrest --stanza=kyc info`; check `repo1-host` |
| **PMM not reachable** | Docker port mapping / firewall | Image listens on 8443 internally → map `-p 443:8443`; open 443 on backup node; iptables FORWARD ACCEPT for 443 |
| **`patronictl restart` hangs** | Interactive prompt | Use `patronictl restart kyc --no-wait` (or `-w` to wait) — never run bare in automation |
| **Deploy fails on a fresh VM** | Missing EPEL/CRB | `dnf install -y epel-release && dnf config-manager --set-enabled crb` before installing pgBackRest deps |

---

## 13. Security Notes — Change These Before Production

⚠️ **All secrets are now managed in `credentials.yaml` (not in playbooks).** The repository ships with `credentials.yaml.example` containing placeholder values. **Copy it to `credentials.yaml`, fill in your real passwords, and encrypt with `ansible-vault` before production use:**

| Variable in `credentials.yaml` | Purpose | Production Value |
|-------------------------------|---------|------------------|
| `postgres_password` | PostgreSQL superuser | **Strong random** |
| `replicator_password` | Streaming replication user | **Strong random** |
| `patroni_admin_password` | Patroni REST API admin | **Strong random** |
| `percona_password` | Percona monitoring user | **Strong random** |
| `pgpool_password` | pgpool monitoring user | **Strong random** |
| `pcp_password` | Pgpool PCP admin | **Strong random** |
| `pmm_admin_password` | PMM web UI admin | **Strong random** |
| `pg_pmm_user_password` | PMM PostgreSQL monitor user | **Strong random** |
| SSH (pgBackRest) | auto-generated keys | Already unique per deployment — store securely |

**Best practices:**

- Use **Ansible Vault** for the entire `credentials.yaml`: `ansible-vault encrypt credentials.yaml` — then run playbooks with `--ask-vault-pass`
- **Never commit `credentials.yaml` to git** — it's in `.gitignore`. Only `credentials.yaml.example` is committed.
- Restrict firewall rules to the **cluster subnet** only
- Never expose etcd (:2379/2380), Patroni REST (:8008), or PostgreSQL (:5432) to the public internet — only the pgpool VIP (:9999) and PMM (:443) should be reachable by application/admin networks
- Put the `pgpass` file somewhere private with `0600` permissions (the playbook uses `/tmp/pgpass0` for bootstrap simplicity — move it after first boot if you prefer)
- Keep PMM behind a VPN or at minimum behind strong auth (change the admin password on first login)

---

## 14. Production Recommendations

1. **Change the VIP interface** — the playbook defaults to `eth0`; verify with `ip link` and set `vip_interface` accordingly (e.g. `ens3`, `enp1s0`).
2. **Tune failover timing** — the defaults (`ttl: 30`, `loop_wait: 10`) give ~40s failover. For faster failover, lower `ttl` to 15–20s (but keep it comfortably above `loop_wait`).
3. **Set `maximum_lag_on_failover` sensibly** — 1 MB (current default) prevents promoting a far-behind replica; consider 100 MB–1 GB for busy workloads so a slightly-lagged replica can still be promoted.
4. **Use `wal_keep_size`** instead of the legacy `wal_keep_segments` on PG 16 if you tune WAL retention manually (Patroni's defaults with slots are fine for most setups).
5. **Separate disks** — put `/postgres/data` and `/var/lib/etcd` on dedicated NVMe/SSD storage; etcd is latency-sensitive.
6. **Automate the first backup** — the playbook intentionally stops at printing the pgBackRest commands. In production, schedule:
   ```bash
   # cron on the backup node
   0 1 * * * sudo -iu postgres pgbackrest --stanza=kyc --type=incr backup
   ```
7. **Test failover monthly** — run [Section 11](#11-failure-testing--learning-by-breaking-things) on a schedule. A HA cluster you never test is a false promise.
8. **Monitor the monitors** — PMM alerting should include: Patroni node down, etcd quorum lost, replica lag, backup age, VIP owner changes.
9. **Keep the deployment reproducible** — the whole point of this repo: one `ansible-playbook` run rebuilds the world. Store your customized `hosts` + vars in Git (secrets in Vault).
10. **Backup the etcd data too** — etcd holds the cluster brain (`/percona_lab/kyc/*`). A full backup strategy includes `etcdctl snapshot save`.

---

## 15. External References

- **Patroni documentation** — https://patroni.readthedocs.io/
- **Percona Distribution for PostgreSQL** — https://www.percona.com/software/postgresql-distribution
- **Percona Patroni setup docs** — https://docs.percona.com/postgresql/16/patroni.html
- **pgpool-II documentation** — https://www.pgpool.net/docs/
- **pgBackRest documentation** — https://pgbackrest.org/
- **PMM (Percona Monitoring & Management)** — https://www.percona.com/software/database-tools/percona-monitoring-and-management
- **etcd documentation** — https://etcd.io/docs/

---

## License

MIT — Adapted from Percona reference architectures and the community HA patterns for PostgreSQL.
