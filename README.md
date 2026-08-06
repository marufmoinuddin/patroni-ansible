# Percona Patroni + Pgpool-II HA Cluster

An Ansible playbook suite for deploying a **3-node high-availability PostgreSQL 16 cluster** with:
- **Patroni** for leader election & automatic failover (backed by etcd)
- **Percona Distribution for PostgreSQL 16**
- **Pgpool-II 4.5** with **Watchdog** for connection pooling, load balancing, and Virtual IP (VIP) management
- **pgBackRest** for backup/restore (dedicated backup node)
- **PMM (Percona Monitoring & Management)** for observability

---

## Architecture

```
┌────────────────────────────────┐
│ Floating VIP — Pgpool leader   │
│ 192.168.122.200  (port 9999)   │
└──────────┬─────────────────────┘
           │
┌──────────┼──────────────────────┐
│          │                      │
▼          ▼                      ▼
┌─────────┐ ┌─────────┐         ┌─────────┐
│  db1    │ │  db2    │         │  db3    │
│ .150    │ │ .151    │         │ .152    │
│         │ │         │         │         │
│ Pgpool  │ │ Pgpool  │         │ Pgpool  │
│ W/Dog   │◄─WD msgs─►│         │ W/Dog   │
│ LEADER  │ │ STANDBY │         │ STANDBY │  ← watchdog leader owns VIP
│         │ │         │         │         │
│ Percona │ │ Percona │         │ Percona │
│ Patroni │ │ Patroni │         │ Patroni │
│Replica  │ │Replica  │         │LEADER   │  ← PostgreSQL leader
│         │ │         │         │         │
│ PG 16   │ │ PG 16   │         │ PG 16   │
│         │ │         │         │         │
│ etcd    │ │ etcd    │         │ etcd    │
└─────────┘ └─────────┘         └─────────┘
           │          │                      │
           └──────────┼──────────────────────┘
                      │
           etcd cluster
```

### Components

| Host | IP | Roles |
|------|-----|-------|
| percona-node-1 | 192.168.122.150 | PostgreSQL + Patroni + etcd + Pgpool-II (Watchdog) |
| percona-node-2 | 192.168.122.151 | PostgreSQL + Patroni + etcd + Pgpool-II (Watchdog) |
| percona-node-3 | 192.168.122.152 | PostgreSQL + Patroni + etcd + Pgpool-II (Watchdog) |
| percona-pgbackrest | 192.168.122.153 | pgBackRest server + PMM Server |

### Network Endpoints

| Service | Port | Description |
|---------|------|-------------|
| **Pgpool-II (VIP)** | **9999** | **Client connection endpoint — connects to Floating VIP 192.168.122.200** |
| Pgpool-II (local) | 9999 | Local Pgpool instance on each node |
| PCP (Pgpool control) | 9898 | Admin/monitoring interface |
| Watchdog heartbeat | 9000 | Inter-node health checks |
| PostgreSQL | 5432 | Direct PostgreSQL (Patroni-managed) |
| Patroni REST API | 8008 | Health checks, cluster status |
| etcd client | 2379 | Patroni ↔ etcd communication |
| etcd peer | 2380 | etcd cluster communication |
| pgBackRest | 8000 | Backup server (if enabled) |
| PMM Server | 443 | Monitoring web UI (HTTPS) |

---

## Prerequisites

- **4 Ubuntu/Debian VMs** with root SSH access
- **Passwordless sudo** or sudo password for `root` user
- **Network connectivity** between all nodes (ports: 2379, 2380, 5432, 8008, 9000, 9898, 9999)
- **VIP 192.168.122.200** must be **unused** and routable on the same subnet
- Network interface for VIP (default: `eth0` — adjust in playbook vars)

---

## Quick Start

```bash
# 1. Clone the repo
git clone https://github.com/marufmoinuddin/patroni-ansible.git
cd patroni-ansible

# 2. Edit inventory (hosts) with your IPs
vim hosts

# 3. Adjust variables in playbooks if needed:
#    - 04_Configure_Pgpool.yml: vip_interface, passwords
#    - 03_Configure_Patroni.yml: passwords, scope
#    - 05_Configure_Pgbackrest.yml: repo path
#    - 06_Install_Pmm_Monitoring.yml: PMM passwords

# 4. Run the full deployment
ansible-playbook -i hosts site.yml --ask-become-pass
```

---

## Playbook Sequence

| # | Playbook | Description |
|---|----------|-------------|
| 01 | `01_Install_Percona.yml` | Locale, purge old packages, install Percona repos, PostgreSQL 16, etcd, Patroni, pgBackRest, Pgpool-II |
| 02 | `02_Configure_Etcd.yml` | Configure 3-node etcd cluster on PostgreSQL nodes |
| 03 | `03_Configure_Patroni.yml` | Create PostgreSQL cluster `kyc`, configure Patroni, start primary → replicas |
| 04 | `04_Configure_Pgpool.yml` | **Install & configure Pgpool-II with Watchdog + VIP (192.168.122.200:9999)** |
| 05 | `05_Configure_Pgbackrest.yml` | Configure pgBackRest server + clients, SSH keys, integrate with Patroni archiving |
| 06 | `06_Install_Pmm_Monitoring.yml` | Deploy PMM Server (Docker), PMM Client + pg_stat_monitor on all PG nodes |

---

## Key Configuration

### Pgpool-II Watchdog + VIP (04_Configure_Pgpool.yml)

```yaml
vars:
  pg_version: "16"
  vip: "192.168.122.200"
  vip_interface: "eth0"        # CHANGE TO MATCH YOUR NIC (e.g., ens3, enp1s0)
  vip_cidr: "24"
  pgpool_port: 9999
  pcp_port: 9898
  wd_port: 9000
  # Passwords — CHANGE IN PRODUCTION!
  pgpool_admin_password: "pgpool_admin_pass"
  pcp_password: "pcp_pass"
  health_check_password: "health_pass"
```

### Patroni Cluster (03_Configure_Patroni.yml)

```yaml
scope: "kyc"
namespace: "percona_lab"
data_dir: "/postgres/data/16/kyc"
# Default passwords — CHANGE!
postgres_password: "qaz123"
replicator_password: "replPasswd"
admin_password: "qaz123"
```

---

## Post-Deployment

### Verify Cluster

```bash
# Check Patroni cluster status
patronictl -c /etc/patroni/patroni.yml list

# Check Pgpool watchdog status
pcp_watchdog_info -h localhost -p 9898 -U pgpool_pcp -w

# Check VIP ownership
ip addr show eth0 | grep 192.168.122.200

# Test connection via VIP
psql -h 192.168.122.200 -p 9999 -U postgres -d postgres
```

### pgBackRest — Manual Steps Required

After playbook 05 completes, run on **pgBackRest server**:

```bash
# 1. Create stanza
sudo -iu postgres pgbackrest --stanza=kyc stanza-create

# 2. Create first full backup
sudo -iu postgres pgbackrest --stanza=kyc --type=full backup

# 3. Verify
sudo -iu postgres pgbackrest --stanza=kyc info
```

### PMM Access

- **URL**: `https://192.168.122.153:443`
- **User**: `admin`
- **Password**: `NewAdminPassword123` (or your custom value)

---

## Failover Behavior

1. **PostgreSQL failover**: Patroni + etcd handle leader election automatically
2. **VIP failover**: Pgpool Watchdog detects leader loss → new watchdog leader claims VIP
3. **Client connections**: Applications connect to VIP:9999 → Pgpool routes to current PostgreSQL leader
4. **Read replicas**: Pgpool load-balances SELECT queries across replicas (configurable)

---

## Customization

### Change Network Interface for VIP

Edit `04_Configure_Pgpool.yml`:
```yaml
vip_interface: "ens3"  # or your interface name
```

### Change PostgreSQL Version

Update `pg_version` in all playbooks and `percona-release setup ppgXX` in 01_Install_Percona.yml.

### Add More Replicas

1. Add host to `[pg_nodes]` in `hosts`
2. Update `etcd` cluster in `02_Configure_Etcd.yml` (static cluster)
3. Re-run playbooks 02-04

### Enable TLS/SSL

- Patroni: Add `ssl: on` + cert paths in `03_Configure_Patroni.yml`
- Pgpool: Set `ssl = on` + certs in the inline `pgpool.conf` block in `04_Configure_Pgpool.yml`
- PMM: Already uses HTTPS (self-signed)

---

## Troubleshooting

| Issue | Check |
|-------|-------|
| VIP not moving | `sudoers` for `ip`/`arping`, `CapabilityBoundingSet` in systemd override |
| Watchdog not forming | Firewall ports 9000 (UDP/TCP), `wd_authkey` match, network connectivity |
| Patroni not starting | `journalctl -u patroni`, etcd health (`etcdctl endpoint health`) |
| Pgpool not connecting | `pool_passwd` MD5 matches, PostgreSQL `pg_hba.conf` allows Pgpool IPs |
| Backup failing | SSH keys exchanged, `pgbackrest` stanza created, repo path writable |

---

## Security Notes

⚠️ **CHANGE ALL DEFAULT PASSWORDS BEFORE PRODUCTION USE:**

- Patroni: `postgres`, `replicator`, `admin` users
- Pgpool: `pgpool_admin`, `pgpool_pcp`, `pgpool_health` users
- PMM: `admin`, `pmm` PostgreSQL user
- pgBackRest: SSH keys (already generated uniquely per deployment)

---

## License

MIT — Adapted from Percona reference architectures.