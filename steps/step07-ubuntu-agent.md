# Step 07 — Enroll Ubuntu Server (VM3)

## Objective
Create a lightweight Ubuntu 22.04 Server VM, configure it on the host-only network, install Elastic Agent via Fleet enrollment, and confirm the agent appears as healthy in the Kibana Fleet UI — making Linux auth logs  available in Elasticsearch.

---

## Prerequisites
- Fleet Server is running and healthy on VM1
- The `linux-endpoints` agent policy exists in Kibana Fleet
- VM1 (ELK + Fleet) is running


---

## VM setup (VMware)

### Specs
| Setting | Value |
|---|---|
| Name | `ubuntutarget` |
| OS | Ubuntu 22.04 LTS Server (64-bit) |
| RAM | 1024 MB |
| CPU | 2 cores |
| Disk | 20 GB (dynamically allocated) |
| Adapter 1 | Host-Only (`VMnet2` — lab network) |

> No GUI, no NAT adapter — this VM is intentionally minimal. It is an SSH target, not a workstation. Keeping it at 1 GB RAM is the right call for an 8 GB host.

> **No NAT adapter needed.** All package installs and agent enrollment happen over the host-only network via VM1. If you need to install packages from the internet, temporarily add a NAT adapter, install, then remove it.

### Ubuntu Server installation notes
- When prompted for additional packages, select **OpenSSH server** — this VM's primary role is as an SSH brute force target

---

## Verify connectivity to VM1

```bash
# Test Fleet Server port
nc -zv 172.31.0.128 8220
# Should show: Connection to 192.168.56.10 8220 port [tcp/*] succeeded

# Test Elasticsearch port
nc -zv 172.31.0.128 9200
# Should show: Connection to 192.168.56.10 9200 port [tcp/*] succeeded
```

If either fails, check VM1's UFW rules are in place  and both VMs are on the same host-only adapter.

---

## Add a swapfile (recommended on 1 GB RAM)

The Elastic Agent process can spike memory during initial enrollment and policy sync. A swapfile prevents the OOM killer from terminating it:

```bash
sudo fallocate -l 1G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Make permanent
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

---

## Get the enrollment command from Kibana

1. In Kibana, go to **Management → Fleet → Agent policies**
2. Click on **linux-endpoints**
3. Click **Add agent** (top right)
4. Select **Enroll in Fleet**
5. Under **What type of host are you enrolling?** → select **DEB x86_64**
6. Copy the full enrollment command — it will look something like:

```bash
curl -L -O https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-8.x.x-linux-x86_64.tar.gz
tar xzvf elastic-agent-8.x.x-linux-x86_64.tar.gz
cd elastic-agent-8.x.x-linux-x86_64
sudo ./elastic-agent install \
  --url=https://192.168.56.10:8220 \
  --enrollment-token=YOUR_TOKEN \
  --insecure
```

---

## Install Elastic Agent on VM3

On VM3, run the full command copied from Kibana:


Expected output:
```
Elastic Agent has been successfully installed.
The agent is running and enrolled to Fleet Server.
```

---

## Verify enrollment

### On VM3 — check the service
```bash
sudo systemctl status elastic-agent
# Should show: active (running)
```

Check agent logs if anything looks wrong:
```bash
sudo journalctl -u elastic-agent -f
# Look for: "Successfully enrolled to Fleet Server"
```

### In Kibana Fleet UI
1. Go to **Management → Fleet → Agents**
2. You should now see three agents:
   - **VM1** — Fleet Server policy — Healthy
   - **VM2** — windows-endpoints policy — Healthy (or offline if suspended)
   - **VM3** — linux-endpoints policy — Healthy
---

## Verify logs arriving in Kibana

1. Go to **Analytics → Discover**
2. Filter for the Ubuntu target:

```kql
agent.name: "ubuntutarget"
```

You should see auth log entries appearing. Test with a specific filter:

```kql
agent.name: "ubuntutarget" AND event.dataset: "system.auth"
# Should show SSH session events from your own login
```

---

## Useful Linux KQL queries (add to cheatsheet)

```kql
# All logs from Ubuntu target
agent.name: "ubuntutarget"

# SSH authentication events
agent.name: "ubuntutarget" AND event.dataset: "system.auth"

# Failed SSH login attempts
agent.name: "ubuntutarget" AND
event.dataset: "system.auth" AND
(system.auth.ssh.event: "Invalid" OR system.auth.ssh.event: "Failed")

# Successful SSH logins
agent.name: "ubuntutarget" AND
event.dataset: "system.auth" AND
system.auth.ssh.event: "Accepted"

```
Added these queries to `resources/kql-cheatsheet.md`.

---

## Key takeaway
Linux endpoints generate a different class of telemetry compared to Windows — there are no event codes, no Sysmon, and no registry. Instead, SSH authentication activity lives in `/var/log/auth.log` and is the primary signal for detecting brute force attacks on Linux targets. Understanding this difference — and knowing which Kibana fields map to Linux auth events (`system.auth.ssh.event`, `source.ip`, `user.name`) versus Windows events (`winlog.event_id`, `winlog.event_data.*`) — is core SOC analyst knowledge that later steps will build on directly.

---

## VM status at end of step

| VM | Status |
|---|---|
| ELK + Fleet (VM1) | ✅ Running — Elasticsearch :9200, Kibana :5601, Fleet :8220 |
| Windows target (VM2) | ⏸ Suspended — resume when needed |
| Ubuntu target (VM3) | ✅ Running — Elastic Agent healthy, auth log flowing |
| Mythic C2 (VM4) | Not yet created |
| osTicket (host) | Not yet installed |

