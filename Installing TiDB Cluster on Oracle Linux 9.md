# Installing TiDB Cluster on Oracle Linux 9 — Updated with Your Hostnames

## Architecture Overview

| Role | IP | Hostname | Components |
|---|---|---|---|
| Control + LB | 10.10.10.17 | router-vm | TiUP, HAProxy, Prometheus, Grafana, Alertmanager |
| Node 1 | 10.10.10.11 | oel9-n1 | PD + TiKV + TiDB |
| Node 2 | 10.10.10.12 | oel9-n2 | PD + TiKV + TiDB |
| Node 3 | 10.10.10.13 | oel9-n3 | PD + TiKV + TiDB |

**Oracle Linux 9 note:** For Oracle Enterprise Linux, TiDB supports the Red Hat Compatible Kernel (RHCK) and does not support the Unbreakable Enterprise Kernel provided by Oracle Enterprise Linux. Confirm you're on RHCK before installing.

---

## Step 1 — Prepare all 4 machines (run on EVERY node)

### 1.1 Switch to Red Hat Compatible Kernel (if on UEK)

```bash
uname -r
# If output contains "uek", switch to RHCK:
sudo dnf install -y kernel
sudo grubby --set-default /boot/vmlinuz-$(rpm -q kernel --qf '%{VERSION}-%{RELEASE}.%{ARCH}\n' | tail -1)
sudo reboot
```

After reboot: `uname -r` should NOT contain `uek`.

### 1.2 Set hostnames

```bash
# On 10.10.10.17:
sudo hostnamectl set-hostname router-vm

# On 10.10.10.11:
sudo hostnamectl set-hostname oel9-n1

# On 10.10.10.12:
sudo hostnamectl set-hostname oel9-n2

# On 10.10.10.13:
sudo hostnamectl set-hostname oel9-n3
```

Verify your `/etc/hosts` on **all four** machines contains:

```
10.10.10.11 oel9-n1
10.10.10.12 oel9-n2
10.10.10.13 oel9-n3
10.10.10.17 router-vm
```

Test from `router-vm`:

```bash
ping -c 2 oel9-n1
ping -c 2 oel9-n2
ping -c 2 oel9-n3
```

### 1.3 Install base packages

```bash
sudo dnf install -y epel-release
sudo dnf install -y numactl tar curl wget net-tools chrony tuned policycoreutils-python-utils
```

### 1.4 Disable firewall (lab) or open required ports (prod)

For a learning setup:

```bash
sudo systemctl stop firewalld
sudo systemctl disable firewalld
```

For production, open these between nodes: `2379, 2380` (PD), `20160, 20180` (TiKV), `4000, 10080` (TiDB), `9090, 3000, 9093, 9100, 9115` (monitoring), `22` (SSH).

### 1.5 Disable SELinux

```bash
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
```

### 1.6 Time sync with chrony

```bash
sudo systemctl enable --now chronyd
chronyc tracking
```

### 1.7 Disable swap

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
echo "vm.swappiness = 0" | sudo tee -a /etc/sysctl.conf
```

### 1.8 Disable Transparent Huge Pages (THP)

```bash
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/defrag

sudo systemctl enable --now tuned
sudo mkdir -p /etc/tuned/balanced-tidb
cat <<EOF | sudo tee /etc/tuned/balanced-tidb/tuned.conf
[main]
include=balanced

[vm]
transparent_hugepages=never

[sysctl]
vm.swappiness=0
fs.file-max=1000000
net.core.somaxconn=32768
net.ipv4.tcp_syncookies=0
EOF
sudo tuned-adm profile balanced-tidb
```

### 1.9 Set file descriptor limits

Add to `/etc/security/limits.conf`:

```
tidb soft nofile 1000000
tidb hard nofile 1000000
tidb soft stack 32768
tidb hard stack 32768
tidb soft core unlimited
tidb hard core unlimited
```

### 1.10 Create the `tidb` user on each node

```bash
sudo useradd tidb
sudo passwd tidb
echo "tidb ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/tidb
```

---

## Step 2 — Set up passwordless SSH from the control machine

Run **only on router-vm (10.10.10.17)** as the `tidb` user:

```bash
sudo su - tidb
ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa

# Copy the key to each node (use hostnames now that /etc/hosts is set):
ssh-copy-id tidb@oel9-n1
ssh-copy-id tidb@oel9-n2
ssh-copy-id tidb@oel9-n3
ssh-copy-id tidb@router-vm

# Verify (no password should be prompted):
ssh tidb@oel9-n1 "hostname"
ssh tidb@oel9-n2 "hostname"
ssh tidb@oel9-n3 "hostname"
```

---

## Step 3 — Install TiUP on the control machine

As `tidb` on `router-vm`:

```bash
curl --proto '=https' --tlsv1.2 -sSf https://tiup-mirrors.pingcap.com/install.sh | sh
source ~/.bashrc

tiup cluster
tiup update --self && tiup update cluster
tiup cluster --version
```

---

## Step 4 — Create the topology file

As `tidb` on `router-vm`, create `~/topology.yaml`:

```yaml
global:
  user: "tidb"
  ssh_port: 22
  deploy_dir: "/tidb-deploy"
  data_dir: "/tidb-data"
  arch: "amd64"

server_configs:
  tidb:
    log.slow-threshold: 300
  tikv:
    readpool.storage.use-unified-pool: true
    readpool.coprocessor.use-unified-pool: true
  pd:
    schedule.leader-schedule-limit: 4
    schedule.region-schedule-limit: 2048
    schedule.replica-schedule-limit: 64

pd_servers:
  - host: 10.10.10.11
  - host: 10.10.10.12
  - host: 10.10.10.13

tidb_servers:
  - host: 10.10.10.11
  - host: 10.10.10.12
  - host: 10.10.10.13

tikv_servers:
  - host: 10.10.10.11
  - host: 10.10.10.12
  - host: 10.10.10.13

monitoring_servers:
  - host: 10.10.10.17

grafana_servers:
  - host: 10.10.10.17

alertmanager_servers:
  - host: 10.10.10.17
```

> Use IPs in `topology.yaml` (not hostnames) — TiUP records exact addresses for component-to-component communication. Hostnames belong in `/etc/hosts` for your convenience; IPs belong here for cluster correctness.

---

## Step 5 — Pre-check and auto-fix the environment

```bash
tiup cluster check ./topology.yaml --user tidb -i ~/.ssh/id_rsa
tiup cluster check ./topology.yaml --apply --user tidb -i ~/.ssh/id_rsa
```

Re-run the first check until you see mostly `Pass`. Some warnings (NUMA, disk type) are fine in a lab.

---

## Step 6 — Deploy the cluster

```bash
tiup cluster deploy tidb-prod v8.5.4 ./topology.yaml --user tidb -i ~/.ssh/id_rsa
```

Confirm with `y`. When complete:

```bash
tiup cluster list
tiup cluster display tidb-prod
```

All components will show `Down/inactive` — expected before first start.

---

## Step 7 — Start the cluster (safe start)

```bash
tiup cluster start tidb-prod --init
```

After safe start, TiUP automatically generates a password for the TiDB root user and returns the password in the command-line interface. **Copy it immediately** — the password is generated only once. If you do not record it or you forgot it, refer to Forget the root password to change the password.

```bash
tiup cluster display tidb-prod
```

All components should be `Up`.

---

## Step 8 — Connect directly to a TiDB node and test

```bash
sudo dnf install -y mysql
mysql -h oel9-n1 -P 4000 -u root -p
# paste the password TiUP gave you
```

Inside:

```sql
SELECT tidb_version()\G
SHOW DATABASES;
CREATE DATABASE test_market;
USE test_market;
CREATE TABLE options_trades (
  id BIGINT PRIMARY KEY AUTO_RANDOM,
  symbol VARCHAR(20),
  strike DECIMAL(10,2),
  expiry DATE,
  trade_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
INSERT INTO options_trades (symbol, strike, expiry) VALUES ('NIFTY', 24500.00, '2026-05-29');
SELECT * FROM options_trades;
```

---

## Step 9 — Install and configure HAProxy on router-vm

```bash
sudo dnf install -y haproxy
sudo systemctl enable haproxy
```

Replace `/etc/haproxy/haproxy.cfg`:

```
global
    log         127.0.0.1 local2
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon
    nbthread    4

defaults
    log                     global
    retries                 2
    timeout connect         2s
    timeout client          30000s
    timeout server          30000s

listen tidb-cluster
    bind 0.0.0.0:3390
    mode tcp
    balance leastconn
    option mysql-check user haproxy_check post-41
    server oel9-n1 10.10.10.11:4000 check inter 2000 rise 2 fall 3
    server oel9-n2 10.10.10.12:4000 check inter 2000 rise 2 fall 3
    server oel9-n3 10.10.10.13:4000 check inter 2000 rise 2 fall 3

listen admin_stats
    bind 0.0.0.0:8080
    mode http
    option httplog
    stats refresh 30s
    stats uri /haproxy
    stats realm HAProxy
    stats auth admin:pingcap123
    stats hide-version
    stats admin if TRUE
```

The minimum version of HAProxy that works with all versions of TiDB is v1.5. Between v1.5 and v2.1, you need to set the post-41 option in mysql-check. It is recommended to use HAProxy v2.2 or newer.

Create the health-check user inside TiDB (from your mysql session in Step 8):

```sql
CREATE USER 'haproxy_check'@'%';
```

Start HAProxy:

```bash
sudo systemctl start haproxy
sudo systemctl status haproxy
```

---

## Step 10 — Connect through HAProxy

From any client:

```bash
mysql -h router-vm -P 3390 -u root -p
# or by IP:
mysql -h 10.10.10.17 -P 3390 -u root -p
```

Check which TiDB node you landed on:

```sql
SELECT @@hostname;
```

Open multiple sessions — you should see `oel9-n1`, `oel9-n2`, `oel9-n3` distributed across them.

HAProxy stats page: `http://router-vm:8080/haproxy` (login `admin` / `pingcap123` — **change this password** in production).

---

## Step 11 — Access the dashboards

- **TiDB Dashboard:** `http://oel9-n1:2379/dashboard` (login: `root` + the password from Step 7) — any of `oel9-n1/n2/n3` works since PD runs on all three
- **Grafana:** `http://router-vm:3000` (default `admin`/`admin`, change on first login)
- **Prometheus:** `http://router-vm:9090`

---

## Useful day-2 commands

```bash
tiup cluster display tidb-prod
tiup cluster stop    tidb-prod
tiup cluster start   tidb-prod          # no --init after the first time
tiup cluster restart tidb-prod
tiup cluster reload  tidb-prod          # apply config changes
tiup cluster scale-out tidb-prod scale.yaml
tiup cluster scale-in  tidb-prod --node 10.10.10.13:4000
tiup cluster upgrade tidb-prod v8.5.5
tiup cluster destroy tidb-prod          # WIPE the cluster
```

---

## Cautions for your lab

1. **Disk** — TiKV is write-heavy. SSDs preferred; HDDs work for learning but expect slow queries.
2. **Resources** — At least 8 GB RAM + 4 CPUs per `oel9-n*`; less and TiKV will OOM.
3. **Quorum** — Don't go below 3 PD or 3 TiKV; you'll lose consensus.
4. **Single-point-of-failure** — `router-vm` is alone. For real HA, run two HAProxy boxes with keepalived/VRRP and a VIP.
5. **Backups** — Learn `br` (Backup & Restore) and `dumpling` once the cluster is healthy.

Want me to walk you through any specific step in more depth — pre-check failures, HAProxy with keepalived, or designing your stock/options schema (partitioning by expiry date, AUTO_RANDOM PK design, TiFlash for analytics)?
