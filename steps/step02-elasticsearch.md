# Step 02 — Install Elasticsearch (VM1)

## Objective
Deploy a single-node Elasticsearch 8.x instance on a Ubuntu 22.04 VMware VM, tune it for a 3 GB RAM constraint, and verify it is reachable from the Windows host browser.

---

## VM setup (VMware)

### Specs
| Setting | Value |
|---|---|
| Name | `ubuntuelkfleet` |
| OS | Ubuntu 22.04 LTS Server (no GUI) |
| RAM | 3072 MB |
| CPU | 2 cores |
| Disk | 50 GB (dynamically allocated) |
| Adapter 1 | NAT (internet — for apt installs) |
| Adapter 2 | Host-Only (`VMnet2` — lab network) |

### Get the host-only IP after boot
```bash
ip a
# Look for the inet address on the host-only adapter
# Note this IP — every other component will talk to this address
# Example: 172.31.0.128
```

---

## Installation

### 1. Download the Elasticsearch package
```bash
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-9.4.2-amd64.deb
```
### 2. Install the package
```bash
dpkg -i elasticsearch-9.4.2-amd64.deb
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
# ── Network ──────────────────────────────────────────────
# Bind to the host-only adapter IP so Kibana and Fleet can reach it
# Replace with your actual host-only IP
network.host: 172.31.0.128
http.port: 9200
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

## Configs saved to repo

The following files (with sensitive values redacted) are saved under `configs/elasticsearch/`:

**`configs/elasticsearch/elasticsearch.yml`**


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
