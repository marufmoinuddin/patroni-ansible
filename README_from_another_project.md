# Patroni + PostgreSQL + pgpool-II High Availability Cluster
## A Complete Manual Deployment and Operations Guide

---

## Table of Contents

1. [Introduction — The Problem Space](#1-introduction--the-problem-space)
2. [Architecture Overview — How the Pieces Fit Together](#2-architecture-overview--how-the-pieces-fit-together)
3. [High Availability Concepts — Building the Mental Model](#3-high-availability-concepts--building-the-mental-model)
4. [Patroni Internals — How the Brain Works](#4-patroni-internals--how-the-brain-works)
5. [pgpool-II — The Traffic Controller](#5-pgpool-ii--the-traffic-controller)
6. [Server Planning — Before You Install Anything](#6-server-planning--before-you-install-anything)
7. [Operating System Preparation](#7-operating-system-preparation)
8. [PostgreSQL Installation & Configuration](#8-postgresql-installation--configuration)
9. [etcd Installation & Cluster Formation](#9-etcd-installation--cluster-formation)
10. [Patroni Installation & Configuration](#10-patroni-installation--configuration)
11. [Initial Cluster Bootstrap](#11-initial-cluster-bootstrap)
12. [Adding Replica Nodes](#12-adding-replica-nodes)
13. [pgpool-II Installation & Configuration](#13-pgpool-ii-installation--configuration)
14. [Operations Guide — Daily Commands](#14-operations-guide--daily-commands)
15. [Failure Testing — Learning by Breaking Things](#15-failure-testing--learning-by-breaking-things)
16. [Troubleshooting Common Issues](#16-troubleshooting-common-issues)
17. [Production Recommendations](#17-production-recommendations)
18. [External References](#18-external-references)

---

## 1. Introduction — The Problem Space

### Who Is This Guide For?

- System administrators who know PostgreSQL basics but have never built a PostgreSQL HA cluster
- Developers who understand `psql` but have never heard of Patroni, etcd, or pgpool-II
- Anyone who wants to deploy PostgreSQL HA **by hand** — no Ansible, no Kubernetes, no magic

**Assumed knowledge:** basic Linux (shell, `systemctl`, editing files with `sudo`), basic PostgreSQL (creating tables, running `psql`). Everything about HA, Patroni, etcd, and pgpool-II is explained from zero.

**The three passwords you will need** (generate them before you start — you will paste them into several config files, and they must be **identical on all nodes**):
1. Patroni REST API password
2. PostgreSQL superuser (`postgres`) password
3. PostgreSQL replication (`replicator`) + monitoring (`pgpool`) passwords

> **A note on hosts:** all examples use `db1/db2/db3` with IPs `192.168.122.150-152` and VIP `192.168.122.200` on CentOS Stream 10 / RHEL-family with `dnf`. The concepts and steps are identical on Debian/Ubuntu — only the package manager differs.

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

PostgreSQL streaming replication is **asynchronous by default** (it can be made synchronous, but that is a different trade-off). The replica receives Write-Ahead Log (WAL) records from the primary and replays them. But PostgreSQL has no built-in mechanism to:

1. **Decide** which replica should become primary when the current one fails
2. **Coordinate** that decision across multiple replicas to prevent "split brain" (two primaries accepting writes)
3. **Notify** applications that the primary has moved
4. **Fence** the old primary so it cannot accept writes after losing leadership

Patroni provides all of this by layering a **consensus-driven state machine** on top of PostgreSQL's native replication.

### The Cast of Characters — Component Roles at a Glance

| Component | Role | What It Does NOT Do |
|-----------|------|---------------------|
| **PostgreSQL** | The actual database engine — stores data, processes queries, streams WAL | Does not decide who is primary; does not fail over automatically |
| **Patroni** | Cluster manager — leader election, failover, replication orchestration, configuration management | Does not route client connections; does not pool connections |
| **etcd** | Distributed consensus store — holds leader lock, cluster state, configuration | Does not manage PostgreSQL; does not understand SQL |
| **pgpool-II** | Connection middleware — routing, pooling, read balancing, Virtual IP (VIP) management | Does not manage PostgreSQL replication; does not elect leaders |
| **Load Balancer (optional, e.g., HAProxy, keepalived)** | Distributes client connections across pgpool-II nodes | Not required if pgpool-II watchdog manages the VIP |

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

### Data Flow Summary

1. **Application** connects to **pgpool-II VIP** (192.168.122.200:9999)
2. **pgpool-II** routes writes to the current **PostgreSQL primary** (detected via streaming replication check)
3. **pgpool-II** optionally load-balances reads across replicas
4. **Patroni** on each PostgreSQL node watches etcd for leadership changes
5. **etcd** holds the **leader lock** — only one Patroni can hold it at a time
6. When the primary fails, **Patroni** on a replica acquires the lock and promotes PostgreSQL
7. **pgpool-II** detects the new primary via health checks and routes traffic there
8. **VIP** moves to the pgpool-II node that is currently the watchdog leader (independent of the PostgreSQL primary)

---

## 3. High Availability Concepts — Building the Mental Model

*If you have never worked with database HA, read this section carefully. These concepts are the foundation for everything that follows.*

### Primary and Replica

- **Primary** (formerly "master"): The single PostgreSQL instance that accepts **reads AND writes**. It generates WAL (Write-Ahead Log) records for every change.
- **Replica** (formerly "slave" / "standby"): A PostgreSQL instance that receives WAL from the primary and replays it. It accepts **reads ONLY** (when `hot_standby = on`).

```
Primary (read/write) ──WAL stream──► Replica (read-only)
       │                                   │
       ▼                                   ▼
  WAL generated                      WAL replayed
```

### Streaming Replication

PostgreSQL's native replication mechanism. The primary continuously sends WAL records to connected replicas over a replication connection (using the `replication` pseudo-database role).

**Key settings on the primary:**
- `wal_level = replica` — includes enough information in WAL for replicas to replay
- `max_wal_senders` — maximum concurrent replication connections
- `max_replication_slots` — replication slots reserve WAL so replicas don't fall behind

**Key settings on the replica:**
- `hot_standby = on` — allows read queries while replaying WAL
- `primary_conninfo` — connection string to the primary (Patroni manages this)

### WAL (Write-Ahead Log)

The **write-ahead log** is PostgreSQL's transaction log. Every data modification is written to WAL **before** it is applied to data files. This ensures durability (crash recovery) and enables replication (replicas replay the same WAL).

**WAL segments** are 16 MB files in `pg_wal/`. They accumulate until:
- A checkpoint completes (data files synced to disk)
- `max_wal_size` is reached (forces a checkpoint)
- Archiving/cleanup removes old segments

### Replication Slots

A **replication slot** is a server-side bookmark that tells the primary: "Do not remove WAL segments until this replica has received them."

Without slots, a slow or disconnected replica could cause the primary to recycle WAL the replica still needs — breaking replication and requiring a full `pg_basebackup` to recover.

Patroni **always uses replication slots** (`use_slots: true` in its configuration).

### Failover vs. Switchover

| Aspect | Failover | Switchover |
|--------|----------|------------|
| **Trigger** | Unplanned — primary crashes, network partition, OOM | Planned — maintenance, OS upgrades, hardware replacement |
| **Initiation** | Automatic (Patroni detects failure) | Manual (`patronictl switchover`) |
| **Data Loss Risk** | Possible (async replication) | Zero (synchronous handoff) |
| **Old Primary** | May still be running (split-brain risk) | Gracefully demoted to replica |
| **Speed** | Seconds to ~40s (depends on TTL) | Near-instant |

### Split Brain

**Split brain** occurs when **two nodes believe they are the primary at the same time** and both accept writes. This corrupts data irrecoverably.

**Causes:**
- Network partition: primary and replica lose contact, both think the other is down
- etcd quorum loss: multiple Patroni nodes think they hold the leader lock
- Clock drift: etcd election timeouts fire incorrectly

**Protection:**
- etcd **quorum requirement** (a majority of nodes must agree)
- Patroni **TTL-based leader lock** (expires if Patroni stops heartbeating)
- **Fencing** (watchdog, STONITH) — forcibly reboots a node that loses quorum
- `maximum_lag_on_failover` — prevents promotion of severely lagged replicas

### Leader Election and Distributed Consensus

Patroni uses **etcd's Raft consensus algorithm** for leader election. Simplified:

1. Each Patroni node tries to **acquire a lock** in etcd: `/service/<scope>/leader`
2. The lock is a **key with a TTL** (time-to-live, e.g., 30 seconds)
3. The node that creates the key becomes **leader**; the others become **followers**
4. The leader must **renew the lock** (heartbeat) before the TTL expires
5. If the leader stops heartbeating (crash, network), the lock expires
6. Followers race to acquire the expired lock — the winner becomes the new leader

**Why etcd?** It provides:
- **Strong consistency** — linearizable reads/writes via Raft
- **TTL/leases** — automatic lock expiration
- **Watch mechanism** — Patroni gets instant notification of changes
- **Cluster membership** — dynamic node discovery

### etcd Quorum

etcd uses Raft, which requires a **majority (quorum)** to operate:

| Cluster Size | Quorum Required | Tolerated Failures |
|--------------|-----------------|---------------------|
| 1 | 1 | 0 (no HA) |
| 3 | 2 | **1** |
| 5 | 3 | 2 |
| 7 | 4 | 3 |

**This is why we use 3 etcd nodes** — it tolerates exactly **one node failure** while maintaining quorum. With 2 nodes, quorum = 2, so **zero failures are tolerated** (which defeats the purpose of HA).

---

## 4. Patroni Internals — How the Brain Works

### How Patroni Starts

When `systemctl start patroni` runs:

1. Patroni reads `/etc/patroni/patroni.yml`
2. Connects to the etcd endpoints listed in `etcd3.hosts`
3. Checks whether a cluster with this `scope` already exists in etcd
4. If **no cluster exists** and this node has `bootstrap.method: initdb`:
   - Runs `initdb` to create a new PostgreSQL data directory
   - Creates users, replication slots, and configuration
   - **Acquires the leader lock** in etcd → becomes primary
5. If **a cluster exists**:
   - Reads the current leader from etcd
   - If **this node was previously primary** and has valid data:
     - Tries to acquire the leader lock
     - If successful → promotes PostgreSQL to primary
     - If another node holds the lock → starts as a replica
   - If **this node was previously a replica** (or is a new node):
     - Checks whether a data directory exists and is valid
     - If not → runs `pg_basebackup` from the current primary
     - Starts PostgreSQL as a replica
     - Registers as a follower in etcd

### How the First PostgreSQL Node Becomes Primary

**The bootstrap process (first node only):**

```
1. Patroni starts on db1
2. Reads patroni.yml → scope: "patroni-cluster", bootstrap.method: initdb
3. Connects to etcd → no existing cluster for this scope
4. Patroni runs initdb:
   - Creates /data/pgdata/16/data with encoding UTF8, locale en_US.UTF-8
   - Enables data checksums (detects corruption)
   - Sets the auth method to scram-sha-256
   - Creates users: postgres (superuser), replicator (replication), pgpool (monitoring)
5. Patroni writes the initial cluster state to etcd:
   - /service/patroni-cluster/leader → {name: "db1", host: "192.168.122.150", ...}
   - /service/patroni-cluster/members/db1 → {role: "leader", ...}
   - /service/patroni-cluster/config → PostgreSQL parameters from bootstrap.dcs.postgresql
   - /service/patroni-cluster/history → timeline history
6. Patroni starts PostgreSQL as PRIMARY
7. Patroni begins heartbeating the leader lock (renews TTL every loop_wait seconds)
```

### How Replicas Join

```
1. Patroni starts on db2
2. Connects to etcd → finds existing cluster, leader = db1
3. Checks local data directory /data/pgdata/16/data
   - If empty or invalid → proceeds to pg_basebackup
4. Runs pg_basebackup:
   - Connects to db1:5432 as replicator user
   - Streams base backup + WAL to the local data directory
   - Creates standby.signal file (tells PostgreSQL to start as a replica)
5. Patroni starts PostgreSQL as REPLICA
6. Patroni registers in etcd: /service/patroni-cluster/members/db2 → {role: "replica", ...}
7. Patroni begins streaming replication from the primary
```

### How Patroni Communicates with etcd

Patroni uses the **etcd v3 API** (via the `python-etcd3` library). Key operations:

| Operation | etcd Key | Purpose |
|-----------|----------|---------|
| **Leader lock** | `/service/<scope>/leader` | Key with TTL; only the leader can write it |
| **Member registration** | `/service/<scope>/members/<name>` | Node metadata, role, state, timeline |
| **Cluster config** | `/service/<scope>/config` | PostgreSQL parameters (dynamic) |
| **Timeline history** | `/service/<scope>/history` | Used by pg_rewind after failover |
| **Watches** | All keys above | Instant notification of changes |

**Heartbeat loop** (runs every `loop_wait` seconds, default 10s):
1. If leader → renew the leader lock TTL
2. If follower → check whether the leader lock has expired
3. Read cluster state from etcd
4. Reconcile local PostgreSQL state with the desired state
5. Apply configuration changes from etcd to postgresql.conf
6. Update local state in etcd

### How Leader Locks Work

The leader lock is a **lease-based key** in etcd:

```yaml
# In patroni.yml bootstrap.dcs:
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
| t=30 | Leader lock **expires** in etcd (TTL reached) |
| t=30-40 | db2 (and db3) next loop iteration → both try to acquire the lock |
| t=30-40 | **One wins** (Raft consensus), becomes the new leader |
| t=30-40 | Winner promotes local PostgreSQL to primary |
| t=30-40 | Winner writes the new leader key to etcd |
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
8. db2 Patroni calls pg_ctl promote (or creates the promote signal file)
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

pgpool-II sits **between applications and PostgreSQL**. It provides:

| Feature | Description |
|---------|-------------|
| **Connection Pooling** | Reuses PostgreSQL connections — reduces the overhead of frequent connect/disconnect |
| **Read/Write Splitting** | Sends `SELECT` to replicas, `INSERT/UPDATE/DELETE` to the primary (optional, via `load_balance_mode`) |
| **Primary Detection** | Uses streaming replication checks (`sr_check`) to determine which backend is primary |
| **Virtual IP (VIP) Management** | The watchdog module manages a floating IP — applications connect to the VIP, not to individual nodes |
| **Failover Detection** | Health checks (`health_check_period`) detect backend failures |
| **PCP (Pgpool Control Protocol)** | Administrative interface for `pcp_*` commands (attach/detach nodes, promote, etc.) |

### What pgpool-II Does NOT Do

| Not pgpool-II's Job | Handled By |
|---------------------|------------|
| PostgreSQL replication management | Patroni |
| Leader election for PostgreSQL | Patroni + etcd |
| `pg_basebackup` / `pg_rewind` | Patroni |
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

**Why this matters:** If pgpool-II tried to manage PostgreSQL failover, you would have **two systems fighting over who is primary**. Patroni is the **single source of truth** for PostgreSQL leadership. pgpool-II **follows** Patroni's lead.

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

**Connection string example:**
```bash
psql -h 192.168.122.200 -p 9999 -U pgpool postgres
# Or from an application: postgresql://pgpool:password@192.168.122.200:9999/postgres
```

---

## 6. Server Planning — Before You Install Anything

### Reference Architecture (3 Nodes for Production)

| Node | Hostname | IP Address | Components |
|------|----------|------------|------------|
| **db1** | db1.example.com | 192.168.122.150 | PostgreSQL 16, Patroni, etcd, pgpool-II |
| **db2** | db2.example.com | 192.168.122.151 | PostgreSQL 16, Patroni, etcd, pgpool-II |
| **db3** | db3.example.com | 192.168.122.152 | PostgreSQL 16, Patroni, etcd, pgpool-II |

Applications connect to the **Virtual IP 192.168.122.200:9999** (served by whichever node is the pgpool-II watchdog leader).

**Why 3 database nodes?**
- PostgreSQL: 1 primary + 2 replicas (tolerates 1 replica loss and still has HA)
- etcd: 3 nodes = quorum of 2 (tolerates 1 etcd node loss)
- pgpool-II: 3 nodes = watchdog quorum of 2 (tolerates 1 pgpool node loss)

### Alternative Topologies

| Topology | Nodes | Use Case | Trade-offs |
|----------|-------|----------|------------|
| **3-node combined** (this guide) | 3 | Standard production | All components on each node; resource contention possible |
| **Dedicated etcd** | 3 DB + 3 etcd | High throughput, strict isolation | More hardware; etcd gets dedicated disks |
| **Dedicated pgpool-II** | 3 DB + 2-3 pgpool | High connection count, separate scaling | More hardware; VIP management separate |
| **2-node + witness** | 2 DB + 1 etcd-only | Limited hardware | **No automatic failover** if 1 DB node fails (etcd quorum lost) |

> **Recommendation:** Start with the 3-node combined topology. Separate components only when you have measured bottlenecks.

### Hardware Sizing Guidelines (Per Node)

| Resource | Minimum | Recommended | Notes |
|----------|---------|-------------|-------|
| **CPU** | 4 vCPU | 8+ vCPU | Patroni + PostgreSQL + etcd + pgpool-II all run here |
| **RAM** | 8 GB | 32+ GB | PostgreSQL `shared_buffers` = 25-40% of RAM; etcd needs low-latency memory |
| **Disk (OS)** | 50 GB | 100 GB | Root filesystem |
| **Disk (PostgreSQL data)** | 100 GB | 500+ GB NVMe | **Separate disk/partition** for `/data/pgdata` — critical for performance |
| **Disk (etcd)** | 10 GB | 50 GB NVMe | **Separate disk** for `/var/lib/etcd` — etcd is latency-sensitive |
| **Network** | 1 Gbps | 10 Gbps | Low latency between nodes is essential for replication and etcd |

### Network Requirements

| Port | Protocol | Source → Destination | Purpose |
|------|----------|---------------------|---------|
| 22 | TCP | Admin → All nodes | SSH |
| 2379 | TCP | All nodes ↔ All nodes | etcd client API (Patroni, pgpool-II) |
| 2380 | TCP | All nodes ↔ All nodes | etcd peer communication (Raft) |
| 5432 | TCP | All nodes ↔ All nodes | PostgreSQL replication + pgpool-II health checks |
| 5432 | TCP | App servers → VIP | Application connections (via pgpool-II) |
| 8008 | TCP | All nodes ↔ All nodes | Patroni REST API (patronictl, callbacks) |
| 9000 | UDP | All nodes ↔ All nodes | pgpool-II watchdog heartbeat |
| 9898 | TCP | Admin → All nodes | PCP (pgpool control protocol) |
| 9999 | TCP | App servers → VIP | pgpool-II client connections |

> **Firewall rule:** Allow all of the above ports **only between cluster nodes** (source = cluster subnet). Do not expose etcd, the Patroni REST API, or PostgreSQL directly to application networks — only the pgpool-II VIP.

### DNS and Hostname Resolution

**All nodes must resolve each other by hostname.** Configure via `/etc/hosts` or internal DNS:

```bash
# /etc/hosts on ALL nodes
192.168.122.150  db1.example.com  db1
192.168.122.151  db2.example.com  db2
192.168.122.152  db3.example.com  db3
192.168.122.200  pgpool-vip.example.com  pgpool-vip
```

**Test:** `ping db1`, `ping db2`, `ping db3` from each node — all must succeed.

---

## 7. Operating System Preparation

*Perform these steps on **ALL 3 nodes** (db1, db2, db3) unless noted otherwise.*

### 7.1 Hostname and /etc/hosts

*(You planned the names and IPs in section 6 — now make them real on each machine.)*

```bash
# On db1:
sudo hostnamectl set-hostname db1.example.com

# On db2:
sudo hostnamectl set-hostname db2.example.com

# On db3:
sudo hostnamectl set-hostname db3.example.com

# On ALL nodes — append to /etc/hosts:
sudo tee -a /etc/hosts <<'EOF'
192.168.122.150  db1.example.com  db1
192.168.122.151  db2.example.com  db2
192.168.122.152  db3.example.com  db3
192.168.122.200  pgpool-vip.example.com  pgpool-vip
EOF
```

**Verify:**
```bash
ping -c 2 db1 && ping -c 2 db2 && ping -c 2 db3
```

### 7.2 Time Synchronization (Critical for etcd)

etcd **requires synchronized clocks**. Clock drift greater than the election timeout causes leadership instability.

```bash
sudo dnf install -y chrony
sudo systemctl enable --now chronyd
```

**Verify:**
```bash
chronyc tracking
chronyc sources -v
```

**Expected:** `System time` offset well under 100ms, several active sources.

### 7.3 Directory Structure and Permissions

```bash
# PostgreSQL data directory (put this on a dedicated disk/partition if possible)
sudo mkdir -p /data/pgdata/16/data
sudo chown -R postgres:postgres /data
sudo chmod 0700 /data/pgdata/16/data

# etcd data directory (put this on a dedicated disk/partition if possible)
sudo mkdir -p /var/lib/etcd/patroni
sudo chown -R etcd:etcd /var/lib/etcd/patroni
sudo chmod 0700 /var/lib/etcd/patroni

# Patroni configuration directory
sudo mkdir -p /etc/patroni
sudo chown postgres:postgres /etc/patroni
sudo chmod 0755 /etc/patroni

# pgpool-II configuration + runtime directories
sudo mkdir -p /etc/pgpool-II /var/run/pgpool /var/log/pgpool
sudo chown -R postgres:postgres /etc/pgpool-II /var/run/pgpool /var/log/pgpool
```

### 7.4 Firewall Configuration

**firewalld example (RHEL/CentOS/Rocky/AlmaLinux). Adjust for UFW/iptables if needed.**

```bash
CLUSTER_SUBNET="192.168.122.0/24"

# PostgreSQL (Patroni manages it, but other nodes must reach it)
firewall-cmd --permanent --zone=public --add-rich-rule="rule family='ipv4' source address='${CLUSTER_SUBNET}' port port='5432' protocol='tcp' accept"

# pgpool-II client port
firewall-cmd --permanent --zone=public --add-rich-rule="rule family='ipv4' source address='${CLUSTER_SUBNET}' port port='9999' protocol='tcp' accept"

# pgpool-II PCP port
firewall-cmd --permanent --zone=public --add-rich-rule="rule family='ipv4' source address='${CLUSTER_SUBNET}' port port='9898' protocol='tcp' accept"

# Patroni REST API
firewall-cmd --permanent --zone=public --add-rich-rule="rule family='ipv4' source address='${CLUSTER_SUBNET}' port port='8008' protocol='tcp' accept"

# pgpool-II watchdog heartbeat (UDP)
firewall-cmd --permanent --zone=public --add-rich-rule="rule family='ipv4' source address='${CLUSTER_SUBNET}' port port='9000' protocol='udp' accept"

# etcd client
firewall-cmd --permanent --zone=public --add-rich-rule="rule family='ipv4' source address='${CLUSTER_SUBNET}' port port='2379' protocol='tcp' accept"

# etcd peer (Raft)
firewall-cmd --permanent --zone=public --add-rich-rule="rule family='ipv4' source address='${CLUSTER_SUBNET}' port port='2380' protocol='tcp' accept"

firewall-cmd --reload
```

**Verify:**
```bash
firewall-cmd --list-all --zone=public
# Look for the rich rules with the correct ports and source subnet
```

### 7.5 System Tuning (Kernel Parameters)

These settings optimize the OS for database workloads.

```bash
# Disable Transparent Huge Pages (THP) — can cause latency spikes in databases
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/defrag

# Make THP disablement persistent across reboots
cat <<'EOF' | sudo tee /etc/rc.d/rc.local
if test -f /sys/kernel/mm/transparent_hugepage/enabled; then
  echo never > /sys/kernel/mm/transparent_hugepage/enabled
  echo never > /sys/kernel/mm/transparent_hugepage/defrag
fi
EOF
sudo chmod +x /etc/rc.d/rc.local

# Adjust vm.swappiness (prefer keeping data in RAM over swapping)
sudo sysctl -w vm.swappiness=1
echo "vm.swappiness=1" | sudo tee /etc/sysctl.d/99-swappiness.conf

# Tune dirty page ratios (control when dirty pages are flushed to disk)
# This prevents I/O spikes by flushing smaller amounts more frequently
sudo sysctl -w vm.dirty_background_ratio=3
sudo sysctl -w vm.dirty_ratio=10
echo "vm.dirty_background_ratio=3" | sudo tee /etc/sysctl.d/99-postgresql.conf
echo "vm.dirty_ratio=10" | sudo tee -a /etc/sysctl.d/99-postgresql.conf

sudo sysctl --system   # Apply all sysctl changes
```

### 7.6 Install Python 3 and venv

Patroni is a Python application and needs a virtual environment.

```bash
sudo dnf install -y python3-pip python3-devel python3-psycopg2
```

---

## 8. PostgreSQL Installation & Configuration

*These steps install PostgreSQL and prepare it for Patroni to manage. Perform them on **all 3 nodes**.*

### 8.1 Add the PostgreSQL YUM Repository

The PostgreSQL Global Development Group (PGDG) provides up-to-date packages.

```bash
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-10-x86_64/pgdg-redhat-repo-latest.noarch.rpm
sudo rpm --import /etc/pki/rpm-gpg/PGDG-RPM-GPG-KEY-RHEL
sudo dnf -qy module disable postgresql
sudo dnf makecache
```

> **Why disable the module?** The base RHEL/CentOS repos ship an older PostgreSQL. Disabling the module prevents conflicts so the PGDG version (16 here) is used.

### 8.2 Install PostgreSQL Server

```bash
sudo dnf install -y postgresql16-server postgresql16-contrib postgresql16-libs
```

This creates:
- The `postgres` OS user (`/usr/pgsql-16/bin`, `/var/lib/pgsql/16` home)
- PostgreSQL binaries in `/usr/pgsql-16/bin`
- The `postgresql-16-setup` helper script

### 8.3 Initialize PostgreSQL — Actually, Let Patroni Do It

**Do NOT run `postgresql-setup initdb` here.** Patroni's `bootstrap` section in `patroni.yml` will run `initdb` itself with the exact flags it needs:

- `auth: scram-sha-256` — strong password-based authentication
- `data-checksums` — detects block corruption (required for `pg_rewind`)
- `encoding: UTF8`
- `locale: en_US.UTF-8`

Letting Patroni run `initdb` guarantees the data directory matches what Patroni expects. Running it manually first would either fail or force Patroni to redo it.

### 8.4 Why `pg_hba.conf` Is Written by Patroni

Patroni **generates and overwrites** `pg_hba.conf` on every start from the `pg_hba:` list in `patroni.yml`. This is one of its biggest conveniences: you never edit `pg_hba.conf` directly, because Patroni owns it.

For understanding, this is what the generated file will look like:

```
# Patroni generates this — do not edit by hand
local   all             all                                     trust
host    all             all             127.0.0.1/32            scram-sha-256
host    all             all             192.168.122.0/24        scram-sha-256
host    replication     replicator      192.168.122.0/24        scram-sha-256
```

Key entries:
- `local all all trust` — Unix-socket connections trusted locally (required for bootstrap and superuser operations)
- `host all all <subnet> scram-sha-256` — remote client connections from the cluster subnet must authenticate
- `host replication replicator <subnet> scram-sha-256` — **replication connections** (used by `pg_basebackup` and streaming) are only allowed for the `replicator` user, from the cluster subnet

### 8.5 The `postgres` User's Sudo Permissions (for VIP Management)

The pgpool-II watchdog moves the Virtual IP by running `ip addr add/del` and `arping` as root. Because pgpool-II runs as the `postgres` user in this guide, that user needs passwordless sudo for exactly these two commands:

```bash
sudo visudo -f /etc/sudoers.d/postgres
# Add this line:
# postgres ALL=(root) NOPASSWD: /usr/sbin/ip, /usr/sbin/arping
# Save and exit.
```

> **Security note:** Grant `NOPASSWD` only to the specific binaries, never to a shell. This lets pgpool-II move the VIP without giving it full root access.

---

## 9. etcd Installation & Cluster Formation

etcd is the **distributed consensus store (DCS)** that Patroni uses for leader election and cluster state. A 3-node etcd cluster survives a single node failure (quorum = 2 of 3).

*Perform these steps on **all 3 nodes**.*

### 9.1 Install etcd

```bash
sudo dnf install -y etcd
```

### 9.2 Configure etcd (Environment File)

Each etcd node needs to know about itself and its peers. Create `/etc/etcd/etcd.conf`. **Replace `db1`, `db2`, `db3` and their IPs with your actual hostnames and IPs.**

**On `db1` (192.168.122.150):**
```bash
cat > /etc/etcd/etcd.conf <<'EOF'
ETCD_NAME=db1
ETCD_DATA_DIR=/var/lib/etcd/patroni
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.122.150:2379"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.122.150:2380"
ETCD_INITIAL_CLUSTER="db1=http://192.168.122.150:2380,db2=http://192.168.122.151:2380,db3=http://192.168.122.152:2380"
ETCD_INITIAL_CLUSTER_STATE=new
ETCD_INITIAL_CLUSTER_TOKEN=patroni-cluster-token
ETCD_PEER_AUTO_TLS="true"
ETCD_PEER_CLIENT_CERT_AUTH="false"
ETCD_LOG_LEVEL="info"
ETCD_ENABLE_V2="false"
EOF
```

**On `db2` (192.168.122.151):**
```bash
cat > /etc/etcd/etcd.conf <<'EOF'
ETCD_NAME=db2
ETCD_DATA_DIR=/var/lib/etcd/patroni
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.122.151:2379"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.122.151:2380"
ETCD_INITIAL_CLUSTER="db1=http://192.168.122.150:2380,db2=http://192.168.122.151:2380,db3=http://192.168.122.152:2380"
ETCD_INITIAL_CLUSTER_STATE=new
ETCD_INITIAL_CLUSTER_TOKEN=patroni-cluster-token
ETCD_PEER_AUTO_TLS="true"
ETCD_PEER_CLIENT_CERT_AUTH="false"
ETCD_LOG_LEVEL="info"
ETCD_ENABLE_V2="false"
EOF
```

**On `db3` (192.168.122.152):**
```bash
cat > /etc/etcd/etcd.conf <<'EOF'
ETCD_NAME=db3
ETCD_DATA_DIR=/var/lib/etcd/patroni
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.122.152:2379"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.122.152:2380"
ETCD_INITIAL_CLUSTER="db1=http://192.168.122.150:2380,db2=http://192.168.122.151:2380,db3=http://192.168.122.152:2380"
ETCD_INITIAL_CLUSTER_STATE=new
ETCD_INITIAL_CLUSTER_TOKEN=patroni-cluster-token
ETCD_PEER_AUTO_TLS="true"
ETCD_PEER_CLIENT_CERT_AUTH="false"
ETCD_LOG_LEVEL="info"
ETCD_ENABLE_V2="false"
EOF
```

**Explanation of every setting:**

| Setting | Meaning |
|---------|---------|
| `ETCD_NAME` | This node's unique name in the cluster. Must match one of the names in `ETCD_INITIAL_CLUSTER`. |
| `ETCD_DATA_DIR` | Where etcd stores its data. Put this on fast storage. |
| `ETCD_LISTEN_CLIENT_URLS` | Which address/port etcd accepts **client** (API) connections on. `0.0.0.0` = all interfaces. |
| `ETCD_ADVERTISE_CLIENT_URLS` | The address **other clients** use to reach this node. Must be the node's real IP, not `0.0.0.0`. |
| `ETCD_LISTEN_PEER_URLS` | Which address/port etcd accepts **peer** (Raft) connections on. |
| `ETCD_INITIAL_ADVERTISE_PEER_URLS` | The address **other etcd nodes** use to reach this node for Raft. |
| `ETCD_INITIAL_CLUSTER` | The full membership list at first startup. **Must be identical on all 3 nodes.** |
| `ETCD_INITIAL_CLUSTER_STATE` | `new` for a fresh cluster; `existing` when joining a running cluster. |
| `ETCD_INITIAL_CLUSTER_TOKEN` | A name for the cluster — prevents accidental merges of two clusters. |
| `ETCD_PEER_AUTO_TLS` | Auto-generate self-signed TLS certs for peer communication. |
| `ETCD_PEER_CLIENT_CERT_AUTH` | Whether peers must present client certificates. `false` here for simplicity. |
| `ETCD_LOG_LEVEL` | Log verbosity. `info` is fine for production. |
| `ETCD_ENABLE_V2` | The legacy v2 API. `false` — Patroni uses v3. |

### 9.3 Start etcd and Verify the Cluster

```bash
sudo systemctl enable --now etcd
sudo systemctl status etcd
```

**Verify cluster health** — run on any one node:

> **What is `etcdctl`?** It is the command-line client for etcd, installed with the `etcd` package (usually `/usr/bin/etcdctl`). `ETCDCTL_API=3` tells it to speak the modern v3 protocol (the same one Patroni uses). Think of it as `psql` for etcd: `endpoint health` asks etcd whether it is alive, `member list` shows the cluster membership.

```bash
ETCDCTL_API=3 etcdctl --endpoints=http://127.0.0.1:2379 endpoint health --cluster
# Expected output: all 3 endpoints report "healthy"
```

Also verify membership:

```bash
ETCDCTL_API=3 etcdctl --endpoints=http://127.0.0.1:2379 member list
# Expected: 3 members, each with a unique name and peer URL
```

> **Troubleshooting:** If a node reports `unhealthy`, re-check `/etc/etcd/etcd.conf` on all nodes (hostnames, IPs, identical `ETCD_INITIAL_CLUSTER`), firewall rules, and that `chronyd` is running everywhere. Restart etcd after any change: `sudo systemctl restart etcd`.

---

## 10. Patroni Installation & Configuration

Patroni is the **cluster manager** that will control PostgreSQL on each node.

*Perform these steps on **all 3 nodes**.*

### 10.1 Install Percona Patroni RPM Packages

```bash
# Install Percona repository
sudo dnf install -y https://repo.percona.com/yum/percona-release-latest.noarch.rpm
sudo percona-release setup ppg-16
sudo rpm --import /etc/pki/rpm-gpg/PERCONA-PPG-16-GPG-KEY

# Install Percona Patroni and PostgreSQL packages
sudo dnf install -y percona-patroni percona-patroni-etcd percona-postgresql16-server percona-postgresql16-contrib percona-postgresql16-libs
```

- `percona-patroni` — Patroni with etcd3 support (binary at `/usr/bin/patroni`)
- `percona-patroni-etcd` — etcd3 client library for Patroni
- `percona-postgresql16-server` — PostgreSQL 16 server
- `percona-postgresql16-contrib` — PostgreSQL contrib extensions
- `percona-postgresql16-libs` — PostgreSQL client libraries

> **Why RPM packages?** Percona provides pre-built, tested RPM packages for Patroni and PostgreSQL. This avoids Python virtual environment management, ensures consistent versions across nodes, and integrates with system package management.

### 10.2 Create the `patroni.yml` Configuration

This is the **core configuration file** for Patroni. Each node gets its own copy with its own `name`, `connect_address`, and `listen` addresses.

**On `db1` (192.168.122.150):**
```bash
cat > /etc/patroni/patroni.yml <<'EOF'
scope: patroni-cluster
name: db1

restapi:
  listen: 0.0.0.0:8008
  connect_address: 192.168.122.150:8008
  authentication:
    username: patroni
    password: CHANGE_ME_PATRONI_RESTAPI_PASSWORD

etcd3:
  hosts:
  - 192.168.122.150:2379
  - 192.168.122.151:2379
  - 192.168.122.152:2379

bootstrap:
  method: initdb
  initdb:
  - auth: scram-sha-256
  - data-checksums
  - encoding: UTF8
  - locale: en_US.UTF-8
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        max_connections: 100
        shared_buffers: 768MB
        effective_cache_size: 2GB
        work_mem: 4MB
        maintenance_work_mem: 128MB
        wal_buffers: 4MB
        max_wal_size: 2GB
        min_wal_size: 512MB
        random_page_cost: 1.1
        max_parallel_workers: 4
        max_parallel_workers_per_gather: 2
        max_worker_processes: 5
        wal_level: replica
        hot_standby: 'on'
        wal_log_hints: 'on'
        max_wal_senders: 5
        max_replication_slots: 5
        wal_keep_size: 512
        checkpoint_completion_target: 0.9
        archive_mode: 'off'
        log_destination: stderr
        logging_collector: 'on'
        log_directory: pg_log
        log_filename: postgresql-%a.log
        log_truncate_on_rotation: 'on'
        log_rotation_age: 1440
        log_rotation_size: 0
        log_line_prefix: '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h'
        log_checkpoints: 'on'
        log_connections: 'on'
        log_disconnections: 'on'
        log_lock_waits: 'on'
        log_temp_files: 0
        log_autovacuum_min_duration: 0
        log_min_duration_statement: 1000
        autovacuum: 'on'
        autovacuum_max_workers: 2
        autovacuum_naptime: '1min'
        autovacuum_vacuum_scale_factor: 0.01
        autovacuum_analyze_scale_factor: 0.01

  users:
    postgres:
      password: CHANGE_ME_POSTGRES_SUPERUSER_PASSWORD
    replicator:
      password: CHANGE_ME_POSTGRES_REPLICATION_PASSWORD
      options: [superuser, replication]
    pgpool:
      password: CHANGE_ME_PGPOOL_PASSWORD
      options: []

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 192.168.122.150:5432
  data_dir: /data/pgdata/16/data
  bin_dir: /usr/pgsql-16/bin
  pgpass: /var/lib/pgsql/16/.pgpass
  authentication:
    replication:
      username: replicator
      password: CHANGE_ME_POSTGRES_REPLICATION_PASSWORD
    superuser:
      username: postgres
      password: CHANGE_ME_POSTGRES_SUPERUSER_PASSWORD
  parameters:
    unix_socket_directories: /var/run/postgresql
  pg_hba:
  - local all all trust
  - host all all 127.0.0.1/32 scram-sha-256
  - host all all 192.168.122.0/24 scram-sha-256
  - host replication replicator 192.168.122.0/24 scram-sha-256

watchdog:
  mode: automatic
  device: /dev/watchdog
  safety: true
EOF
```

**On `db2` (192.168.122.151):** same file, but change these lines:

```yaml
name: db2
# in restapi:
  connect_address: 192.168.122.151:8008
# in postgresql:
  connect_address: 192.168.122.151:5432
```

**On `db3` (192.168.122.152):** same file, but change these lines:

```yaml
name: db3
# in restapi:
  connect_address: 192.168.122.152:8008
# in postgresql:
  connect_address: 192.168.122.152:5432
```

**Set correct ownership and permissions:**
```bash
sudo chown postgres:postgres /etc/patroni/patroni.yml
sudo chmod 0640 /etc/patroni/patroni.yml
```

### 10.4 Explaining Every Section of `patroni.yml`

**`scope:`** — The name of this cluster. All nodes must use the same value. It becomes part of the etcd key prefix (`/service/<scope>/...`) so multiple clusters can share one etcd.

**`name:`** — This node's unique member name. Must differ on each node. Used in etcd member keys and by `patronictl`.

**`restapi:`** — Patroni's own HTTP API on port 8008. Used by `patronictl`, monitoring, and the role-change callback script. `listen` binds all interfaces; `connect_address` tells others which address to use. Authentication protects the API so random hosts can't query/modify cluster state.

**`etcd3:`** — Where Patroni finds etcd. List **all 3** endpoints so Patroni keeps working if one etcd node is down.

**`bootstrap:`** — Instructions used **only when creating a brand-new cluster**:
- `method: initdb` — create the cluster from scratch (alternative: `pg_basebackup` to clone from an existing cluster)
- `initdb:` — flags passed to the `initdb` command. `data-checksums` is required for `pg_rewind`.
- `dcs:` — cluster-level defaults stored in etcd. These are shared by ALL nodes (unlike per-node settings above):
  - `ttl: 30` — leader lock lifetime in seconds (failover speed = TTL + loop_wait)
  - `loop_wait: 10` — how often Patroni checks/renews state
  - `retry_timeout: 10` — how long to retry etcd operations before giving up
  - `maximum_lag_on_failover: 1048576` — max replication lag (in bytes) a replica may have to be eligible for promotion
  - `postgresql.use_pg_rewind: true` — lets a demoted old primary resync from the new primary using `pg_rewind` (fast, instead of full `pg_basebackup`)
  - `postgresql.use_slots: true` — use replication slots so WAL is never recycled too early
  - `postgresql.parameters:` — PostgreSQL settings Patroni manages cluster-wide. Patroni writes these into every node's `postgresql.conf` and keeps them consistent.
- `users:` — PostgreSQL roles created at bootstrap:
  - `postgres` — the superuser (used for Patroni's local management and by operators)
  - `replicator` — has `superuser, replication` attributes; used for `pg_basebackup` and streaming replication
  - `pgpool` — plain login role used by pgpool-II for health checks and `SHOW POOL_NODES`

**`postgresql:`** — Per-node PostgreSQL settings:
- `listen` / `connect_address` — bind address and the address other nodes use to reach this node
- `data_dir` — where PostgreSQL data lives (must match what you created in section 7.3)
- `bin_dir` — path to PostgreSQL binaries (PGDG installs to `/usr/pgsql-16/bin`)
- `pgpass` — where Patroni stores the replication password file (used by `pg_basebackup`)
- `authentication:` — credentials Patroni uses for replication and superuser operations
- `parameters:` — extra per-node settings (here: Unix socket directory)
- `pg_hba:` — the `pg_hba.conf` content Patroni will generate on every node. **This is the single source of truth for access control.**

**`watchdog:`** — Optional but strongly recommended. Patroni uses a Linux watchdog device: if a node loses contact with etcd (possible split-brain), the kernel forcibly reboots it after a timeout. `mode: automatic` lets Patroni use `/dev/watchdog` if present. `safety: true` means Patroni refuses to run without the watchdog once configured.

### 10.5 Password Management (CRITICAL)

Replace every `CHANGE_ME_*` placeholder with strong, randomly generated passwords:

```bash
# Example generation (run once, save outputs securely in a password manager):
openssl rand -base64 24   # repeat for each password
```

Keep them consistent **across all 3 nodes** — the same passwords must appear in every node's `patroni.yml`, otherwise replicas cannot authenticate to the primary.

### 10.6 Enable the Percona Patroni Systemd Service

The Percona RPM provides a systemd service unit at `/usr/lib/systemd/system/percona-patroni.service`:

```ini
[Unit]
Description=Runners to orchestrate a high-availability PostgreSQL
After=etcd.service
Wants=etcd.service

[Service]
Type=simple
User=postgres
Group=postgres
ExecStart=/usr/bin/patroni /etc/patroni/patroni.yml
KillMode=process
KillSignal=SIGINT
TimeoutStartSec=0
TimeoutStopSec=60
Restart=on-abnormal
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Enable and start it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable percona-patroni
```

> **Note:** Do not start Patroni yet. Bootstrap order matters — see the next section.

---

## 11. Initial Cluster Bootstrap

Now we create the cluster. **Order matters more than anything else in this guide.**

### 11.1 Start Patroni on the First Node Only (db1)

```bash
# On db1 ONLY:
sudo systemctl start percona-patroni
sudo systemctl status percona-patroni
```

**Watch what happens** (in another terminal):
```bash
sudo journalctl -u percona-patroni -f
```

You should see:
1. Patroni connecting to etcd
2. `initdb` being executed (creating the data directory)
3. The `postgres`, `replicator`, and `pgpool` users being created
4. PostgreSQL starting
5. Patroni acquiring the leader lock → `db1` becomes **Leader**

**Verify leadership:**
```bash
# On db1 (or any node — patronictl talks to etcd):
/usr/bin/patronictl -c /etc/patroni/patroni.yml list
```

> **What is `patronictl`?** It is Patroni's own command-line tool (installed at `/usr/bin/patronictl` by the Percona RPM). It reads the same `patroni.yml`, asks etcd (and the REST API) about cluster state, and issues cluster commands. `list` is the command you will run most often — it prints a one-line summary per node. Other useful subcommands: `switchover`, `failover`, `edit-config`, `restart`, `reload`. Run `patronictl --help` to see them all.

Expected output:
```
+ Cluster: patroni-cluster (id: ...) +----+-----------+
| Member | Host           | Role    | State   | TL | Lag in MB |
+--------+----------------+---------+---------+----+-----------+
| db1    | 192.168.122.150| Leader  | running |  1 |         0 |
+--------+----------------+---------+---------+----+-----------+
```

**What just happened internally:**
1. Patroni on db1 saw no existing cluster in etcd for `scope: patroni-cluster`
2. It ran `initdb` with the flags from `bootstrap.initdb`
3. It created the three users from `bootstrap.users`
4. It wrote the cluster config and the leader key into etcd
5. It started PostgreSQL as the primary

> **Why start only one node?** If two empty nodes started at the same time, both would try to run `initdb` and create the cluster — a race that can produce two conflicting "first primaries". Starting one node guarantees a single clean bootstrap. The others join afterwards.

---

## 12. Adding Replica Nodes

### 12.1 Start Patroni on the Remaining Nodes

```bash
# On db2 ONLY:
sudo systemctl start percona-patroni
sudo systemctl status percona-patroni

# Then on db3 ONLY:
sudo systemctl start percona-patroni
sudo systemctl status percona-patroni
```

**Watch the logs** (`sudo journalctl -u percona-patroni -f`) on each node. You should see:
1. Patroni connects to etcd and finds leader = db1
2. Local data directory is empty → Patroni runs `pg_basebackup` from db1
3. `standby.signal` is created (PostgreSQL will start in recovery mode)
4. PostgreSQL starts as a **Replica**
5. Patroni registers the member in etcd

> **Note:** The first `pg_basebackup` can take a while depending on data size and network speed. Be patient — no manual intervention is needed.

### 12.2 Verify Full Cluster Health

Run on **any node**:
```bash
/usr/bin/patronictl -c /etc/patroni/patroni.yml list
```

Expected output:
```
+ Cluster: patroni-cluster (id: ...) +----+-----------+
| Member | Host            | Role    | State   | TL | Lag in MB |
+--------+-----------------+---------+---------+----+-----------+
| db1    | 192.168.122.150 | Leader  | running |  1 |         0 |
| db2    | 192.168.122.151 | Replica | running |  1 |         0 |
| db3    | 192.168.122.152 | Replica | running |  1 |         0 |
+--------+-----------------+---------+---------+----+-----------+
```

**Reading the columns:**
- `Member` — node name from `patroni.yml`
- `Host` — the `connect_address` of that node
- `Role` — `Leader` (accepts writes) or `Replica` (read-only)
- `State` — `running` = healthy. Other values: `stopped`, `creating replica`, `initializing new cluster`
- `TL` — **timeline**. Increments on every promotion. All nodes should normally be on the same timeline.
- `Lag in MB` — replication lag. `0` is healthy; a growing number means the replica cannot keep up.

### 12.3 Verify Replication Directly

```bash
# On the leader (db1), as the postgres OS user:
sudo -u postgres /usr/pgsql-16/bin/psql -c "SELECT * FROM pg_stat_replication;"
```

You should see two rows (db2 and db3) with `state = streaming` and increasing `sent_lsn`/`replay_lsn` values.

**Sanity test — create a table on the leader, check it appears on a replica:**
```bash
# On db1 (leader):
sudo -u postgres /usr/pgsql-16/bin/psql -c "CREATE TABLE ha_test (id serial primary key, ts timestamptz default now()); INSERT INTO ha_test DEFAULT VALUES;"

# On db2 (replica):
sudo -u postgres /usr/pgsql-16/bin/psql -c "SELECT * FROM ha_test;"
# Should return the row inserted on db1
```

---

## 13. pgpool-II Installation & Configuration

pgpool-II is the **access plane**: it pools connections, routes reads/writes, and manages the Virtual IP via its watchdog module.

*Perform these steps on **all 3 nodes**.*

### 13.1 Add the Pgpool-II YUM Repository and Install

```bash
sudo dnf install -y https://www.pgpool.net/yum/rpms/4.7/redhat/rhel-10-x86_64/pgpool-II-release-4.7-1.noarch.rpm
sudo rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-PGPOOL-II
sudo dnf makecache
sudo dnf install -y pgpool-II-pg16 iputils sudo
```

- `pgpool-II-pg16` — pgpool-II built against PostgreSQL 16
- `iputils` — provides `arping`, used when moving the VIP
- `sudo` — the `postgres` user needs it to run `ip`/`arping` for VIP management (configured in section 8.5)

### 13.2 Deploy the External Lifecheck Script

pgpool-II's watchdog needs to verify whether a peer pgpool node is truly alive. The default method (`heartbeat`) only proves the machine is up — a machine can be up while pgpool-II has died. The **external** method runs a command on the peer; here we check whether PostgreSQL is actually accepting connections on that node.

```bash
sudo cat > /usr/local/bin/pgpool-lifecheck.sh <<'EOF'
#!/bin/bash
# External lifecheck script for Pgpool-II watchdog
# Checks if PostgreSQL is accepting connections on a given host
PG_HOST="${1:-}"

if [ -z "$PG_HOST" ]; then
  echo "Usage: $0 <postgresql_host>"
  exit 1
fi

# pg_isready return codes:
# 0 = accepting connections, 1 = rejecting, 2 = no response, 3 = no attempt
/usr/pgsql-16/bin/pg_isready -h "$PG_HOST" -p 5432 -t 2
EOF
sudo chmod 0755 /usr/local/bin/pgpool-lifecheck.sh
sudo chown postgres:postgres /usr/local/bin/pgpool-lifecheck.sh
```

### 13.3 Configure `pgpool.conf`

This is pgpool-II's main configuration file. **Replace IPs and `CHANGE_ME_PGPOOL_PASSWORD` with your values.**

**On `db1` (192.168.122.150):**
```bash
sudo cat > /etc/pgpool-II/pgpool.conf <<'EOF'
listen_addresses = '*'
port = 9999
socket_dir = '/var/run/pgpool'
pcp_listen_addresses = '*'
pcp_port = 9898
pcp_socket_dir = '/var/run/pgpool'

backend_hostname0 = '192.168.122.150'
backend_port0 = 5432
backend_weight0 = 1
backend_data_directory0 = '/data/pgdata/16/data'
backend_flag0 = 'ALLOW_TO_FAILOVER'

backend_hostname1 = '192.168.122.151'
backend_port1 = 5432
backend_weight1 = 1
backend_data_directory1 = '/data/pgdata/16/data'
backend_flag1 = 'ALLOW_TO_FAILOVER'

backend_hostname2 = '192.168.122.152'
backend_port2 = 5432
backend_weight2 = 1
backend_data_directory2 = '/data/pgdata/16/data'
backend_flag2 = 'ALLOW_TO_FAILOVER'

enable_pool_hba = on
pool_passwd = 'pool_passwd'
num_init_children = 8
max_pool = 10
child_life_time = 300
child_max_connections = 0
connection_life_time = 300
client_idle_limit = 0
log_child_processes = on

connect_timeout = 10000
socket_timeout = 30000
pcp_connect_timeout = 10
pcp_idle_timeout = 60

authentication_timeout = 60

pid_file_name = '/var/run/pgpool/pgpool.pid'
logdir = '/var/log/pgpool'

connection_cache = on

health_check_period = 10
health_check_timeout = 10
health_check_user = 'pgpool'
health_check_password = 'CHANGE_ME_PGPOOL_PASSWORD'
health_check_max_retries = 3
health_check_retry_delay = 3

sr_check_period = 5
sr_check_user = 'pgpool'
sr_check_password = 'CHANGE_ME_PGPOOL_PASSWORD'
delay_threshold = 10485760

load_balance_mode = on
ignore_leading_white_space = on
read_only_function_list = 'pg_is_in_recovery,pg_last_wal_receive_lsn,pg_last_wal_replay_lsn,pg_last_xact_replay_timestamp'

master_slave_mode = on
master_slave_sub_mode = 'stream'

# Pgpool does NOT perform failover. Patroni is the sole authority.
failover_command = ''
follow_primary_command = ''
failover_on_backend_error = off
search_primary_node_timeout = 10

use_watchdog = on
wd_listen_address = '192.168.122.150'
wd_listen_port = 9000

hostname0 = '192.168.122.150'
wd_port0 = 9000
pgpool_port0 = 9999

hostname1 = '192.168.122.151'
wd_port1 = 9000
pgpool_port1 = 9999

hostname2 = '192.168.122.152'
wd_port2 = 9000
pgpool_port2 = 9999

wd_lifecheck_method = 'external'
wd_interval = 3
wd_life_point = 5
wd_lifecheck_external_command = '/usr/local/bin/pgpool-lifecheck.sh'
wd_lifecheck_external_argument = '%h'
wd_lifecheck_external_timeout = 10

failover_when_quorum_exists = on
failover_require_consensus = on
enable_consensus_with_half_votes = on

delegate_IP = '192.168.122.200'
if_cmd_path = '/usr/bin'
if_up_cmd = '/usr/bin/sudo /usr/sbin/ip addr add $_IP_$/24 dev enp3s0 label enp3s0:0'
if_down_cmd = '/usr/bin/sudo /usr/sbin/ip addr del $_IP_$/24 dev enp3s0'
arping_cmd = '/usr/bin/sudo /usr/sbin/arping -U $_IP_$ -w 1 -I enp3s0'

log_destination = 'stderr'
logging_collector = on
log_directory = '/var/log/pgpool'
log_filename = 'pgpool-%a.log'
log_truncate_on_rotation = on
log_rotation_age = 1440
log_rotation_size = 0
log_connections = on
log_hostname = on
log_statement = off
log_per_node_statement = off
log_error_verbosity = default
log_client_messages = on
log_standby_delay = 'none'

memory_cache_enabled = off
EOF
```

**On `db2` (192.168.122.151):** same file with **one** difference — `wd_listen_address = '192.168.122.151'`.

**On `db3` (192.168.122.152):** same file with **one** difference — `wd_listen_address = '192.168.122.152'`.

**Permissions:**
```bash
sudo chown postgres:postgres /etc/pgpool-II/pgpool.conf
sudo chmod 0644 /etc/pgpool-II/pgpool.conf
```

### 13.4 Explaining the `pgpool.conf` Settings

**Listeners and PCP:**
- `listen_addresses = '*'`, `port = 9999` — where clients connect
- `pcp_port = 9898` — the Pgpool Control Protocol port (admin commands like `pcp_attach_node`)
- `socket_dir` / `pcp_socket_dir` — Unix socket locations

**Backend definitions (the PostgreSQL servers behind pgpool-II):**
- `backend_hostname0/1/2` — the three PostgreSQL nodes. pgpool-II tries them in order (0 first) when looking for the primary.
- `backend_weightN` — load-balancing weight (reads are distributed proportionally; 1:1:1 = even)
- `backend_data_directoryN` — pgpool-II uses this to detect whether the backend was the last primary (for its internal `pg_rewind`-like logic)
- `backend_flag0 = 'ALLOW_TO_FAILOVER'` — permits pgpool-II to mark this backend down if health checks fail

**Connection pooling:**
- `num_init_children = 8` — number of pgpool-II worker processes; **each** can hold up to `max_pool` connections to PostgreSQL. Total possible backend connections = 8 × 10 = 80 (mind `max_connections = 100` on PostgreSQL!)
- `max_pool = 10` — max cached connections per child process
- `child_life_time = 300` — recycle child processes after 300s (releases stale connections)
- `connection_life_time = 300` — recycle cached client connections
- `connection_cache = on` — enable pooling

**Health checks (does PostgreSQL answer?):**
- `health_check_period = 10` — check every 10 seconds
- `health_check_user/password` — the `pgpool` role we created at bootstrap
- `health_check_max_retries = 3`, `health_check_retry_delay = 3` — mark a backend DOWN only after 3 failed checks, 3s apart (avoids flapping)

**Streaming replication checks (who is the primary?):**
- `sr_check_period = 5` — ask PostgreSQL "are you in recovery?" every 5s
- `sr_check_user/password` — the same `pgpool` role
- `delay_threshold = 10485760` — replicas with more than 10 MB lag are excluded from read load-balancing

**Read/write splitting:**
- `master_slave_mode = on`, `master_slave_sub_mode = 'stream'` — pgpool-II treats backends as a streaming-replication cluster: one primary + replicas
- `load_balance_mode = on` — SELECTs go to replicas (weighted), writes go to the primary
- `read_only_function_list` — functions (like `pg_is_in_recovery()`) that prove a query is read-only, so pgpool-II can safely route them to replicas

**Failover commands — deliberately EMPTY:**
- `failover_command = ''`, `follow_primary_command = ''` — pgpool-II must NOT promote or reattach backends by itself
- `failover_on_backend_error = off` — don't detach a backend on a single query error
- `search_primary_node_timeout = 10` — when pgpool-II starts, it looks for the primary; give up after 10s

> **Why empty?** Patroni is the authority on PostgreSQL leadership. If pgpool-II also ran failover commands, you'd have two systems fighting. pgpool-II only **follows**: it health-checks, it reads `pg_is_in_recovery()`, and it routes accordingly. (A role-change callback keeps it in sync even on graceful switchovers — see section 16.1.)

**Watchdog (VIP management):**
- `use_watchdog = on` — enable the watchdog module
- `wd_listen_address` / `wd_listen_port = 9000` — this node's watchdog listening address (UDP)
- `hostname0/1/2` + `wd_portN` + `pgpool_portN` — the watchdog cluster membership list. **Must list all 3 nodes identically.**
- `wd_lifecheck_method = 'external'` — verify peers by running a command (our lifecheck script) instead of blind heartbeats
- `wd_life_point = 5`, `wd_interval = 3` — a peer is considered dead after 5 missed checks, 3s apart (~15s)
- `failover_when_quorum_exists = on` + `failover_require_consensus = on` + `enable_consensus_with_half_votes = on` — only move the VIP when a quorum of watchdog nodes agrees (prevents split-brain VIP)

**Virtual IP:**
- `delegate_IP = '192.168.122.200'` — the VIP applications connect to
- `if_up_cmd` / `if_down_cmd` — commands to add/remove the VIP (`$_IP_$` is substituted; adjust `enp3s0` to your interface name)
- `arping_cmd` — announce the VIP on the network after moving it (so switches update their ARP tables)

### 13.5 Configure `pool_hba.conf`

Client authentication for pgpool-II itself (before it forwards to PostgreSQL):

```bash
sudo cat > /etc/pgpool-II/pool_hba.conf <<'EOF'
# TYPE  DATABASE   USER    ADDRESS             METHOD
local   all        all                          trust
host    all        all     127.0.0.1/32         scram-sha-256
host    all        all     192.168.122.0/24     scram-sha-256
EOF
sudo chown postgres:postgres /etc/pgpool-II/pool_hba.conf
sudo chmod 0640 /etc/pgpool-II/pool_hba.conf
```

### 13.6 Create `pool_passwd`

pgpool-II uses this file to verify client passwords against PostgreSQL (when `enable_pool_hba = on`, it needs the pool users' passwords to connect to backends):

```bash
sudo cat > /etc/pgpool-II/pool_passwd <<'EOF'
pgpool:CHANGE_ME_PGPOOL_PASSWORD
EOF
sudo chown postgres:postgres /etc/pgpool-II/pool_passwd
sudo chmod 0640 /etc/pgpool-II/pool_passwd
```

### 13.7 Create `pcp.conf`

PCP (Pgpool Control Protocol) authenticates with a username:MD5-hash pair. Generate the hash with pgpool's own tool:

> **What is `pg_md5`?** A small utility shipped with pgpool-II that computes the MD5 hash of a password in the format PostgreSQL/pgpool expects. (It is not related to `md5sum`.) The command prints a warning line first and the hash on the second line, which is why the snippet takes line 2 (`sed -n 2p`) and keeps only the hash token (`awk '{print $NF}'`).

```bash
PGPOOL_PASSWORD="CHANGE_ME_PGPOOL_PASSWORD"
PGPOOL_MD5_HASH=$(/usr/pgsql-16/bin/pg_md5 "$PGPOOL_PASSWORD" | sed -n 2p | awk '{print $NF}')

sudo cat > /etc/pgpool-II/pcp.conf <<EOF
pgpool:${PGPOOL_MD5_HASH}
EOF
sudo chown postgres:postgres /etc/pgpool-II/pcp.conf
sudo chmod 0640 /etc/pgpool-II/pcp.conf
```

### 13.8 Create `pgpool_node_id`

Each node needs a unique numeric ID so the watchdog cluster knows who is who (0, 1, 2):

```bash
# On db1:
echo "0" | sudo tee /etc/pgpool-II/pgpool_node_id
# On db2:
echo "1" | sudo tee /etc/pgpool-II/pgpool_node_id
# On db3:
echo "2" | sudo tee /etc/pgpool-II/pgpool_node_id

# On all three:
sudo chown postgres:postgres /etc/pgpool-II/pgpool_node_id
sudo chmod 0644 /etc/pgpool-II/pgpool_node_id
```

### 13.9 Create a Systemd Override for Fast Shutdown

pgpool-II's default shutdown can be slow (it waits for children). This override makes `systemctl stop pgpool` finish quickly, which matters during failover testing:

```bash
sudo mkdir -p /etc/systemd/system/pgpool.service.d
sudo cat > /etc/systemd/system/pgpool.service.d/override.conf <<'EOF'
[Service]
TimeoutStopSec=10
KillMode=mixed
EOF
sudo systemctl daemon-reload
```

### 13.10 Start pgpool-II on All Nodes

```bash
# On db1:
sudo systemctl enable --now pgpool
sudo systemctl status pgpool

# On db2:
sudo systemctl enable --now pgpool
sudo systemctl status pgpool

# On db3:
sudo systemctl enable --now pgpool
sudo systemctl status pgpool
```

> **Tip:** Start them one at a time and check the log between each: `sudo journalctl -u pgpool -f`. The first node to start becomes the initial watchdog leader and takes the VIP.

### 13.11 Verify the Watchdog Quorum

```bash
pcp_watchdog_info -h 127.0.0.1 -U pgpool -p 9898 -w -d
```

Expected output (example):
```
3 3 YES 192.168.122.150:9999 Linux db1
```

Reading it: `3` watchdog nodes total, `3` in quorum, `YES` = quorum achieved, then the current watchdog **leader** (which owns the VIP).

### 13.12 Verify the VIP

```bash
# On ALL 3 nodes:
ip addr show enp3s0 | grep 192.168.122.200 || echo "VIP not on this node"
```

Exactly **one** node should show the VIP — that is the current watchdog leader.

### 13.13 End-to-End Test Through pgpool-II

```bash
# From any node (or an app server), through the VIP:
PGPASSWORD="CHANGE_ME_PGPOOL_PASSWORD" psql -h 192.168.122.200 -p 9999 -U pgpool postgres -c "SHOW POOL_NODES;"
```

Expected output shows all 3 backends; one has `role = primary`, the others `role = standby` and `pg_role = replica`. The columns are:

| Column | Meaning |
|--------|---------|
| `node_id` | Backend number (0, 1, 2 — matches `backend_hostname0/1/2`) |
| `host` / `port` | The PostgreSQL backend address |
| `status` | `up` (healthy) or `down` (pgpool-II stopped sending it traffic) |
| `role` | `primary` or `standby` — what pgpool-II believes about the backend |
| `pg_role` | `primary` or `replica` — what PostgreSQL reports (`pg_is_in_recovery()`) |

```bash
# Create data through the pool, read it back:
PGPASSWORD="CHANGE_ME_PGPOOL_PASSWORD" psql -h 192.168.122.200 -p 9999 -U pgpool postgres -c "CREATE TABLE app_test AS SELECT generate_series(1,10) AS n;"
PGPASSWORD="CHANGE_ME_PGPOOL_PASSWORD" psql -h 192.168.122.200 -p 9999 -U pgpool postgres -c "SELECT count(*) FROM app_test;"
```

---

## 14. Operations Guide — Daily Commands

### 14.1 Check Patroni Cluster State

```bash
/usr/bin/patronictl -c /etc/patroni/patroni.yml list
```

**Expected output:**
```
+ Cluster: patroni-cluster (id: ...) +----+-----------+
| Member | Host            | Role    | State   | TL | Lag in MB |
+--------+-----------------+---------+---------+----+-----------+
| db1    | 192.168.122.150 | Leader  | running | 12 |         0 |
| db2    | 192.168.122.151 | Replica | running | 12 |         0 |
| db3    | 192.168.122.152 | Replica | running | 12 |         0 |
+--------+-----------------+---------+---------+----+-----------+
```

**What to look for:** exactly one `Leader`, all `State = running`, all `TL` equal, `Lag in MB = 0` (or small and stable).

### 14.2 Check pgpool-II Backend Status

```bash
PGPASSWORD="CHANGE_ME_PGPOOL_PASSWORD" psql -h 192.168.122.200 -p 9999 -U pgpool postgres -c "SHOW POOL_NODES;"
```

**Expected:** one backend `role = primary`, others `role = standby` / `pg_role = replica`, all `status = up`.

### 14.3 Check etcd Cluster Health

```bash
ETCDCTL_API=3 etcdctl --endpoints=http://127.0.0.1:2379 endpoint health --cluster
# All endpoints: healthy
```

### 14.4 Check the VIP Location

```bash
ip addr show enp3s0 | grep 192.168.122.200
```

### 14.5 View Patroni Logs

```bash
sudo journalctl -u percona-patroni -f
```

### 14.6 View pgpool-II Logs

```bash
sudo journalctl -u pgpool -f
```

### 14.7 View etcd Logs

```bash
sudo journalctl -u etcd -f
```

### 14.8 Query PostgreSQL Directly (bypassing pgpool-II)

```bash
# On the primary (db1) as the OS user:
sudo -u postgres /usr/pgsql-16/bin/psql -c "SELECT pg_is_in_recovery();"
# false = primary, true = replica

# Replication status on the primary:
sudo -u postgres /usr/pgsql-16/bin/psql -c "SELECT * FROM pg_stat_replication;"
```

### 14.9 Check Patroni REST API (programmatic monitoring)

```bash
# Cluster-wide view (JSON):
curl -s -u patroni:CHANGE_ME_PATRONI_RESTAPI_PASSWORD http://192.168.122.150:8008/cluster | python3 -m json.tool

# Local node status:
curl -s -u patroni:CHANGE_ME_PATRONI_RESTAPI_PASSWORD http://192.168.122.150:8008/patroni | python3 -m json.tool
```

### 14.10 Changing PostgreSQL Parameters After Deployment

The parameters you put in `bootstrap.dcs.postgresql.parameters` live in **etcd** (they are cluster-wide), not in each node's `postgresql.conf`. Patroni propagates them to every node. To change them later:

```bash
/usr/bin/patronictl -c /etc/patroni/patroni.yml edit-config
```

This opens the cluster config in your `$EDITOR`. Edit the `postgresql.parameters` section, save, and Patroni reloads the new settings on all nodes automatically (it will tell you which parameters need a `restart` vs a `reload` — `restart` values require PostgreSQL to restart, which Patroni can do for you with `patronictl restart`).

> **Never edit `postgresql.conf` by hand on a Patroni-managed node.** Patroni owns it and will overwrite your changes on the next loop. The correct way is always `patronictl edit-config`.

---

## 15. Failure Testing — Learning by Breaking Things

**Test in a controlled window. Every test below assumes you have a safe way to recover (start the service again).**

### 15.1 Primary Node Failure (Patroni Failover)

**Goal:** Prove that killing the primary promotes a replica automatically.

**Steps:**

1. Identify the current primary:
   ```bash
   patronictl -c /etc/patroni/patroni.yml list
   # Assume db1 is Leader
   ```

2. **On db1:** simulate a crash by stopping Patroni (it stops PostgreSQL too):
   ```bash
   sudo systemctl stop patroni
   ```

3. **On db2 or db3:** watch the failover:
   ```bash
   sudo journalctl -u patroni -f
   # In a second terminal:
   watch -n 2 'patronictl -c /etc/patroni/patroni.yml list'
   ```

4. **Observe:** within ~30-40 seconds (TTL 30s + loop_wait 10s), one replica becomes `Leader`. The leader lock expired in etcd, and the surviving replica won the election.

5. **Verify pgpool-II routing:**
   ```bash
   PGPASSWORD="CHANGE_ME_PGPOOL_PASSWORD" psql -h 192.168.122.200 -p 9999 -U pgpool postgres -c "SHOW POOL_NODES;"
   # The 'primary' role should now point to the newly promoted node
   ```

6. **Recover db1:** bring it back — Patroni will use `pg_rewind` to resync from the new primary and rejoin as a replica:
   ```bash
   # On db1:
   sudo systemctl start patroni
   # Verify with patronictl list — all 3 nodes back, one leader
   ```

> **What you just proved:** automatic failover works, pgpool-II follows the new primary, and the old primary can rejoin safely.

> **What the application experienced:** any query that was mid-flight when the primary died returned an error (connection reset, `server closed the connection unexpectedly`). Queries sent **after** failover completed succeeded against the new primary. This is normal — applications must be written to **retry failed transactions** (or use a pooler that does). HA makes the outage short; it does not make it invisible.

### 15.2 Network Failure (Partition)

**Goal:** Understand what happens when nodes cannot reach each other but nothing crashed.

**Simulation options:**
- `sudo iptables -A INPUT -s 192.168.122.151 -j DROP` on db1 (blocks db2's traffic only)
- Or disable the network interface briefly on one node

**What happens:**

*If the **primary** is isolated from etcd (but not from clients):*
- The leader lock expires (no heartbeats reach etcd)
- A replica is promoted by etcd quorum
- The old primary still accepts client writes **until it notices it lost the lock** — Patroni's `safety` watchdog (if enabled) can force-reboot the node to prevent split-brain

*If a **replica** is isolated:*
- etcd keeps quorum (2 of 3 still reachable)
- The replica stays a replica; when the network returns, it resyncs

> **Key takeaway:** Network partitions are the hardest failure mode. The etcd quorum + Patroni lock TTL + watchdog are your protections. Always test partitions in a lab before production.

### 15.3 Replica Node Failure

**Goal:** Show that a replica loss is a non-event.

**Steps:**
1. On a replica (say db2): `sudo systemctl stop patroni`
2. `patronictl list` → db2 shows `State: stopped`, db1 still Leader, db3 still Replica
3. Clients keep working — pgpool-II simply stops load-balancing reads to db2
4. Restart: `sudo systemctl start patroni` → db2 rejoins, resyncs from primary, `Lag` returns to 0

### 15.4 pgpool-II Watchdog Leader Failure (VIP Failover)

**Goal:** Prove the VIP moves to another pgpool-II node.

**Steps:**
1. Find the watchdog leader: `pcp_watchdog_info -h 127.0.0.1 -U pgpool -p 9898 -w -d`
2. **On that node:** stop pgpool-II: `sudo systemctl stop pgpool`
3. **On another node:** watch the VIP:
   ```bash
   watch -n 1 'ip addr show enp3s0 | grep 192.168.122.200 || echo "VIP not on this node"'
   ```
4. **Observe:** within ~5-15s (lifecheck interval × life points), the VIP appears on another node
5. Verify clients can still reach `192.168.122.200:9999`
6. Restart pgpool-II on the failed node — it rejoins as a **standby** watchdog and does NOT steal the VIP

### 15.5 Manual Switchover (Planned Maintenance)

**Goal:** Perform a controlled, zero-downtime role change (e.g., before OS upgrades).

**Steps:**
1. Identify current primary (db1) and a candidate replica (db2)
2. Run switchover:
   ```bash
   /usr/bin/patronictl -c /etc/patroni/patroni.yml switchover
   # Follow the prompts: cluster name (patroni-cluster), primary (db1), candidate (db2)
   ```
3. Watch `/usr/bin/patronictl list` — db2 becomes `Leader`, db1 becomes `Replica`
4. Verify pgpool-II: `SHOW POOL_NODES` should show db2 as primary
   - **If it still shows the old primary for writes**, you have hit the graceful-switchover routing problem — see section 16.1 for the permanent fix

> **Why switchover instead of just failing over?** Switchover is graceful: Patroni waits for the replica to catch up, then hands over with **zero data loss**. Failover is reactive and can lose the last few async transactions.

---

## 16. Troubleshooting Common Issues

### 16.1 pgpool-II Still Sending Traffic to the Old Primary After a Graceful Switchover

**Symptom:** After `patronictl switchover`, `SHOW POOL_NODES` still lists the old primary as `primary`, and writes fail with `cannot execute UPDATE in a read-only transaction`.

**Cause:** pgpool-II's `sr_check` detects **crashed** primaries. In a graceful switchover, the old primary becomes a *healthy replica* — it still answers queries, so pgpool-II never sees a backend failure and never re-runs primary discovery. `failover_command`/`follow_primary_command` only fire on `DOWN` events, which don't happen here.

**Permanent fix — Patroni `on_role_change` callback:**
Patroni can run a script whenever a node's role changes. The script tells **every** pgpool-II instance which node is the new primary via PCP (`pcp_promote_node`), then re-attaches the healthy replicas.

1. **Create the callback script on ALL 3 nodes:**
   ```bash
   sudo cat > /usr/local/bin/pgpool-patroni-rolechange.sh <<'EOF'
   #!/bin/bash
   # Patroni on_role_change callback: invoked as <action> <role> <cluster_name>
   # Only acts when THIS node is promoted to primary. Syncs pgpool routing on
   # ALL pgpool nodes to Patroni's new leader via PCP (pcp_promote_node),
   # then re-attaches the healthy replicas (promote marks others down internally).
   set -uo pipefail

   ACTION="${1:-}"
   ROLE="${2:-}"
   CLUSTER="${3:-}"

   PCPPASSFILE="/var/lib/pgsql/16/.pcppass"
   PCP_USER="pgpool"
   PCP_PORT="9898"
   PGPOOL_NODES="192.168.122.150 192.168.122.151 192.168.122.152"
   LOG="/var/log/pgpool/patroni-rolechange.log"

   # Static host -> backend node_id mapping (must match pgpool.conf backend_hostnameN)
   declare -A NID=( [192.168.122.150]=0 [192.168.122.151]=1 [192.168.122.152]=2 )

   log() { echo "$(date -Is) [$(hostname)] action=$ACTION role=$ROLE cluster=$CLUSTER $*" >> "$LOG"; }

   if [ "$ROLE" != "primary" ]; then
     exit 0
   fi

   log "Promotion detected — syncing pgpool routing via PCP"

   LEADER_HOST=""
   RAW=$(curl -sf --max-time 3 http://localhost:8008/cluster 2>/dev/null || echo "")
   if [ -n "$RAW" ]; then
     LEADER_HOST=$(echo "$RAW" | python3 -c "
   import sys, json
   try:
       for m in json.load(sys.stdin).get('members', []):
           if m.get('role') == 'leader':
               print(m.get('host', ''))
               break
   except Exception:
       pass
   " 2>/dev/null || echo "")
   fi
   [ -z "$LEADER_HOST" ] && LEADER_HOST=$(hostname -I | awk '{print $1}')
   LEADER_NID="${NID[$LEADER_HOST]:-}"

   if [ -z "$LEADER_NID" ]; then
     log "ERROR: cannot map leader host $LEADER_HOST to a backend node_id — aborting"
     exit 1
   fi

   log "Patroni leader=$LEADER_HOST node=$LEADER_NID — promoting on all pgpool nodes"

   FAILURES=0
   for PG in $PGPOOL_NODES; do
     out=$(PCPPASSFILE="$PCPPASSFILE" pcp_promote_node -h "$PG" -U "$PCP_USER" -p "$PCP_PORT" -w -n "$LEADER_NID" 2>&1)
     rc=$?
     if [ $rc -eq 0 ]; then
       log "OK  promote node=$LEADER_NID on pgpool $PG: $out"
     else
       log "ERR promote node=$LEADER_NID on pgpool $PG (rc=$rc): $out"
       FAILURES=$((FAILURES+1))
     fi
   done

   sleep 2
   for PG in $PGPOOL_NODES; do
     for N in 0 1 2; do
       out=$(PCPPASSFILE="$PCPPASSFILE" pcp_attach_node -h "$PG" -U "$PCP_USER" -p "$PCP_PORT" -w "$N" 2>&1)
       rc=$?
       if [ $rc -eq 0 ]; then
         log "OK  attach node=$N on pgpool $PG"
       else
         log "WARN attach node=$N on pgpool $PG (rc=$rc): $out"
       fi
     done
   done

   log "Role-change sync complete (promote failures=$FAILURES)"
   exit 0
   EOF
   sudo chmod 0755 /usr/local/bin/pgpool-patroni-rolechange.sh
   sudo chown postgres:postgres /usr/local/bin/pgpool-patroni-rolechange.sh
    ```

2. **Create `.pcppass` for the `postgres` user** (so the script can run `pcp_*` non-interactively):
   ```bash
   PGPOOL_PASSWORD="CHANGE_ME_PGPOOL_PASSWORD"
   sudo mkdir -p /var/lib/pgsql/16
   sudo cat > /var/lib/pgsql/16/.pcppass <<EOF
   *:*:pgpool:${PGPOOL_PASSWORD}
   localhost:9898:pgpool:${PGPOOL_PASSWORD}
   127.0.0.1:9898:pgpool:${PGPOOL_PASSWORD}
   192.168.122.150:9898:pgpool:${PGPOOL_PASSWORD}
   192.168.122.151:9898:pgpool:${PGPOOL_PASSWORD}
   192.168.122.152:9898:pgpool:${PGPOOL_PASSWORD}
   EOF
   sudo chown postgres:postgres /var/lib/pgsql/16/.pcppass
   sudo chmod 0600 /var/lib/pgsql/16/.pcppass
   ```
   **File format:** one `host:port:username:password` entry per line — the same idea as PostgreSQL's `.pgpass`. The first `*:*:pgpool:...` line is a wildcard fallback for any host/port, so PCP commands succeed no matter which target they are aimed at. The file must be owned by `postgres` with mode `0600`, otherwise PCP refuses to read it.

3. **Register the callback in Patroni** (once — it propagates to all nodes via etcd):
   ```bash
   /usr/bin/patronictl -c /etc/patroni/patroni.yml edit-config
   # Add under the postgresql: section:
   #   callbacks:
   #     on_role_change: /usr/local/bin/pgpool-patroni-rolechange.sh
   # Save and exit. Patroni reloads this on all nodes automatically.
   ```

4. **Test:** run `/usr/bin/patronictl switchover` again — `SHOW POOL_NODES` must update almost instantly.

### 16.2 Patroni Cannot Connect to etcd

**Symptoms:** Patroni logs show etcd connection errors; `patronictl list` fails or shows an empty cluster.

**Causes & fixes:**
| Cause | Check / Fix |
|-------|-------------|
| Firewall blocks 2379/2380 | `firewall-cmd --list-all`; re-apply rules from section 7.4 |
| Wrong `etcd3.hosts` in `patroni.yml` | Verify IPs and ports match `ETCD_ADVERTISE_CLIENT_URLS` |
| etcd not running | `systemctl status etcd`; `systemctl start etcd` |
| etcd unhealthy | `etcdctl endpoint health --cluster`; check `journalctl -u etcd -f` |
| Clock drift | `chronyc tracking`; fix NTP |

### 16.3 Replica Cannot Join Cluster (pg_basebackup errors)

**Symptoms:** Patroni logs on the new replica show `pg_basebackup` failures.

**Causes & fixes:**
| Cause | Check / Fix |
|-------|-------------|
| Firewall blocks 5432 | Verify rich rules (section 7.4) |
| `pg_hba.conf` missing replication entry | It is generated from `pg_hba:` in `patroni.yml` — make sure the `host replication replicator <subnet>` line exists |
| Wrong `replicator` password | Must match `bootstrap.users.replicator.password` AND `postgresql.authentication.replication.password` on ALL nodes |
| Primary not reachable | `ping db1`; `psql -h db1 -U replicator -d postgres -c 'select 1'` |

### 16.4 Split-Brain Protection Issues

**Symptoms:** Patroni logs show nodes competing for leadership; multiple nodes try to be primary.

**Causes & fixes:**
| Cause | Check / Fix |
|-------|-------------|
| etcd quorum lost (2+ etcd nodes down) | Restore the down nodes immediately; quorum = 2 of 3 |
| Clock drift | Synchronize all nodes with the same NTP source |
| Network partition | Verify all nodes can reach etcd peers and each other |

**Emergency note:** If quorum is lost, etcd **stops accepting writes** — Patroni cannot renew locks. No automatic failover will happen until quorum is restored. This is by design (fail-safe, not fail-some).

### 16.5 Replication Lag Is High

**Symptoms:** `patronictl list` shows high `Lag in MB`; `pg_stat_replication` shows large `write_lag/flush_lag/replay_lag`.

**Causes & fixes:**
| Cause | Check / Fix |
|-------|-------------|
| Slow network | Check bandwidth/latency between nodes |
| Replica disk I/O bottleneck | Monitor replica disk (iostat, iotop) |
| Primary overloaded | Watch CPU, WAL generation rate |
| `max_wal_senders` / `wal_keep_size` too small | Increase in `bootstrap.dcs.postgresql.parameters` (or `patronictl edit-config`), then reload |

### 16.6 Failed Bootstrap (Two Nodes Both Tried to Initialize)

**Symptom:** Cluster state is inconsistent; more than one node thinks it created the cluster.

**Fix:** Clean slate — on ALL nodes stop Patroni, remove the data directory and etcd keys, then re-bootstrap **one node at a time**:
```bash
sudo systemctl stop patroni
sudo rm -rf /data/pgdata/16/data
# Only if the etcd keys are garbage — delete the whole scope:
ETCDCTL_API=3 etcdctl --endpoints=http://127.0.0.1:2379 del /service/patroni-cluster --prefix
# Then start db1 first (section 11), wait for Leader, then db2, then db3
```

> **Never delete etcd keys casually** — that is the cluster's brain. Only do it when you are deliberately re-bootstrapping and accept losing cluster state.

---

## 17. Production Recommendations

### 17.1 Backups

Patroni gives you HA — **it is not a backup**. HA protects against node loss, not against:
- Accidental `DELETE FROM users` (replicates to all replicas!)
- Corruption that replicates
- Ransomware

- **Full backups:** `pg_basebackup` or better: **pgBackRest** / **WAL-G** (support PITR and S3/object storage)
- **WAL archiving:** enable `archive_mode = on` + `archive_command` in `patronictl edit-config` so you can recover to any point in time
- **Test restores regularly.** A backup you have never restored is not a backup.

### 17.2 Monitoring and Alerting

| Layer | What to Watch |
|-------|---------------|
| PostgreSQL | `pg_stat_activity`, `pg_stat_replication`, lag, connections, deadlocks |
| Patroni | Leader state via REST API (`/cluster`), failover events, member health |
| etcd | `etcdctl endpoint health`, leader changes, fsync latency |
| pgpool-II | `SHOW POOL_NODES`, watchdog quorum, pool utilization |
| OS | CPU, RAM, disk I/O, network on every node |

Alert on: primary down, role change (unexpected), replication lag > threshold, etcd quorum loss, disk usage > 80%, connection saturation.

### 17.3 TLS/SSL

This guide uses HTTP/plaintext for simplicity and readability. **In production:**
- PostgreSQL: `ssl = on`, cluster CA + per-node certs; update `pg_hba` to `hostssl`
- Patroni REST API: terminate TLS (reverse proxy) or use `restapi` HTTPS options
- etcd: replace `ETCD_PEER_AUTO_TLS` with a real CA (`ETCD_CERT_FILE`, `ETCD_KEY_FILE`, `ETCD_TRUSTED_CA_FILE`), client certs for Patroni
- pgpool-II: `ssl = on` for client connections; pool_passwd supports AES-encrypted entries

### 17.4 Separate etcd Disks

etcd commits every write to disk with `fsync` (that's how it guarantees consensus). Slow disks mean slow elections and higher failover latency. Put `/var/lib/etcd` on its own NVMe/SSD, away from PostgreSQL's data disk and OS logs.

### 17.5 Fencing / Watchdog

The watchdog (configured in `patroni.yml`) is your last line of defense against split-brain: a node that loses contact with etcd gets **force-rebooted by the kernel** instead of continuing to serve writes as a "zombie primary". Ensure:
- Kernel module loaded: `modprobe softdog` (software watchdog) or a hardware watchdog
- `/dev/watchdog` exists before Patroni starts
- Test it: fail a node and confirm it reboots

### 17.6 Connection Pooling in Applications

pgpool-II pools connections, but applications should still use their own connection pooler (e.g., HikariCP, psycopg pool, PgBouncer in front of pgpool-II if needed). Reason: application-level pools give you control over timeouts, validation, and retry behaviour during failover, and they protect pgpool-II from connection storms.

### 17.7 Test Failover Regularly

Schedule monthly chaos drills:
1. Kill the primary (`systemctl stop patroni`) — verify failover under 60s
2. Kill the pgpool watchdog leader — verify VIP moves
3. Run a graceful switchover — verify zero-downtime
4. Recover the failed node — verify rejoin with `pg_rewind`
5. Record the observed failover time and compare with your SLO

### 17.8 Configuration Hygiene

- Store `patroni.yml`, `pgpool.conf` in Git (secrets via a secrets manager, not in the repo)
- Version your schema changes (e.g., with `migra` or a migration tool)
- Set `archive_mode = on` in production even if you don't do PITR yet — flipping it later requires a restart

---

## 18. External References

- **Patroni Documentation:** https://patroni.readthedocs.io/en/latest/
- **Pgpool-II Documentation:** https://www.pgpool.net/docs/latest/en/index.html
- **etcd Documentation:** https://etcd.io/docs/current/
- **PostgreSQL Streaming Replication:** https://www.postgresql.org/docs/current/warm-standby.html
- **pgBackRest (recommended backup tool):** https://pgbackrest.org/
- **Percona Patroni Guide (reference for production hardening):** https://www.percona.com/blog/patroni-guide/

---

## Appendix A — Quick Reference: Key Files and Commands

| Item | Location / Command |
|------|--------------------|
| Patroni config | `/etc/patroni/patroni.yml` |
| PostgreSQL data | `/data/pgdata/16/data` |
| PostgreSQL binaries | `/usr/pgsql-16/bin/` |
| etcd config | `/etc/etcd/etcd.conf` |
| etcd data | `/var/lib/etcd/patroni` |
| pgpool-II config | `/etc/pgpool-II/pgpool.conf` |
| pgpool-II auth files | `/etc/pgpool-II/pool_hba.conf`, `pool_passwd`, `pcp.conf` |
| Cluster status | `patronictl -c /etc/patroni/patroni.yml list` |
| Manual switchover | `patronictl -c /etc/patroni/patroni.yml switchover` |
| etcd health | `ETCDCTL_API=3 etcdctl --endpoints=http://127.0.0.1:2379 endpoint health --cluster` |
| pgpool backends | `psql -h <VIP> -p 9999 -U pgpool postgres -c "SHOW POOL_NODES;"` |
| Watchdog status | `pcp_watchdog_info -h 127.0.0.1 -U pgpool -p 9898 -w -d` |

## Appendix B — Service Management Cheat Sheet

```bash
# Patroni (do NOT start/stop PostgreSQL directly — Patroni owns it)
sudo systemctl start|stop|restart|status patroni

# etcd
sudo systemctl start|stop|restart|status etcd

# pgpool-II
sudo systemctl start|stop|restart|status pgpool

# Safe cluster-wide restart sequence:
# 1. Stop pgpool-II on all nodes (clients lose access briefly)
# 2. Stop patroni on replicas, then on the primary (or do a graceful switchover first)
# 3. Start patroni on the primary, wait for Leader
# 4. Start patroni on replicas
# 5. Start pgpool-II on all nodes
```

---

*This guide is written for operators who want to understand every layer of a PostgreSQL HA stack before trusting it in production. Read the concepts first, deploy in the order given, and test every failure scenario before you need it.*
