# Step 03 — Install Elasticsearch (VM1)

## Objective
Deploy a single-node Elasticsearch 8.x instance on a Ubuntu 22.04 VirtualBox VM, tune it for a 3 GB RAM constraint, and verify it is reachable from the Windows host browser.

---

## VM setup (VirtualBox)

### Specs
| Setting | Value |
|---|---|
| Name | `soc-elk` |
| OS | Ubuntu 22.04 LTS Server (no GUI) |
| RAM | 3072 MB |
| CPU | 2 cores |
| Disk | 50 GB (dynamically allocated) |
| Adapter 1 | NAT (internet — for apt installs) |
| Adapter 2 | Host-Only (`vboxnet0` — lab network) |

### Get the host-only IP after boot
```bash
ip a
# Look for the inet address on the host-only adapter (usually enp0s8)
# Note this IP — every other component will talk to this address
# Example: 192.168.56.10
```

---

## Installation

### 1. Update the system
```bash
sudo apt update && sudo apt upgrade -y
```

### 2. Install prerequisites
```bash
sudo apt install -y curl wget gnupg apt-transport-https
```

### 3. Add the Elastic GPG key and repository
```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch \
  | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] \
  https://artifacts.elastic.co/packages/8.x/apt stable main" \
  | sudo tee /etc/apt/sources.list.d/elastic-8.x.list
```

### 4. Install Elasticsearch
```bash
sudo apt update && sudo apt install -y elasticsearch
```

> **Important:** The installer prints a built-in `elastic` superuser password and an enrollment token during first install. **Copy both immediately** — they are not shown again. Store them somewhere safe (a local text file outside the repo).

---

## Configuration

### Edit `elasticsearch.yml`
```bash
sudo nano /etc/elasticsearch/elasticsearch.yml
```

Key changes to make:

```yaml
# ── Cluster ──────────────────────────────────────────────
cluster.name: soc-lab

# ── Node ─────────────────────────────────────────────────
node.name: elk-node-1

# ── Network ──────────────────────────────────────────────
# Bind to the host-only adapter IP so Kibana and Fleet can reach it
# Replace with your actual host-only IP
network.host: 192.168.56.10
http.port: 9200

# ── Discovery (single-node) ──────────────────────────────
discovery.type: single-node

# ── Security (enabled by default in 8.x) ─────────────────
xpack.security.enabled: true
xpack.security.enrollment.enabled: true
```

> All other settings can remain at their defaults for this lab.

### Cap the JVM heap (critical on 8 GB host)

Create a custom JVM options file so the default auto-sizing does not claim too much RAM:

```bash
sudo nano /etc/elasticsearch/jvm.options.d/heap.options
```

Add:
```
-Xms1g
-Xmx1g
```

This limits Elasticsearch to 1 GB heap, leaving the rest of the VM's 3 GB for Kibana, Fleet Server, and the OS.

### Add a swapfile (safety net against OOM kills)
```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Make permanent across reboots
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

Verify swap is active:
```bash
free -h
# Swap line should show 2.0G total
```

---

## Start and enable the service

```bash
sudo systemctl daemon-reload
sudo systemctl enable elasticsearch
sudo systemctl start elasticsearch
```

Check status:
```bash
sudo systemctl status elasticsearch
# Should show: active (running)
```

Check logs if it fails to start:
```bash
sudo journalctl -u elasticsearch -f
```

---

## Firewall rules

Allow port 9200 from the host-only subnet only — never expose Elasticsearch directly to the internet:

```bash
sudo ufw allow from 192.168.56.0/24 to any port 9200
sudo ufw allow from 192.168.56.0/24 to any port 9300
sudo ufw allow OpenSSH
sudo ufw enable
sudo ufw status
```

---

## Verification

### From inside the VM
```bash
curl -k -u elastic:YOUR_PASSWORD https://localhost:9200
```

Expected response:
```json
{
  "name" : "elk-node-1",
  "cluster_name" : "soc-lab",
  "cluster_uuid" : "...",
  "version" : {
    "number" : "8.x.x",
    ...
  },
  "tagline" : "You Know, for Search"
}
```

### From the Windows host browser
```
https://192.168.56.10:9200
```

Accept the self-signed certificate warning. Enter `elastic` and your password when prompted. You should see the same JSON response.

---

## Configs saved to repo

The following files (with sensitive values redacted) are saved under `configs/elasticsearch/`:

**`configs/elasticsearch/elasticsearch.yml`**
```yaml
cluster.name: soc-lab
node.name: elk-node-1
network.host: REDACTED
http.port: 9200
discovery.type: single-node
xpack.security.enabled: true
xpack.security.enrollment.enabled: true
```

**`configs/elasticsearch/heap.options`**
```
-Xms1g
-Xmx1g
```

---

## Challenges faced

<!-- Fill this in as you work through the step -->
_Document any errors encountered and how they were resolved. Examples:_
- _Port 9200 unreachable from host → forgot to configure the host-only adapter IP in `network.host` (was still `localhost`)_
- _Elasticsearch failed to start → JVM out of memory, resolved by adding the heap.options file before first start_
- _curl returned SSL error → used `-k` flag to bypass self-signed cert in lab environment_

---

## Key takeaway

Elasticsearch defaults are designed for multi-node production clusters. For a single-node lab on constrained hardware, three things matter most: binding to the correct network interface (not `localhost`), capping the JVM heap so the OS and co-located services have RAM to breathe, and never exposing port 9200 to anything beyond the lab subnet — Elasticsearch has no rate limiting and an open 9200 is a critical misconfiguration in any real environment.

---

## VM status at end of step

| VM | Status |
|---|---|
| ELK + Fleet (VM1) | ✅ Running — Elasticsearch active on :9200 |
| Windows target (VM2) | Not yet created |
| Ubuntu target (VM3) | Not yet created |
| Mythic C2 (VM4) | Not yet created |
| osTicket (host) | Not yet installed |
