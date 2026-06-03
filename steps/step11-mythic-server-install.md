# Step 11 — Install Mythic C2 Server (VM4)

## Objective
Deploy a Mythic C2 server on a new Ubuntu 22.04 VMware VM, install all prerequisites, clone the Mythic repository, start the Mythic service, and confirm the web UI is accessible — establishing the attacker infrastructure needed for the C2 simulation.

---

## Prerequisites
- Attack diagram is drawn and the planned attack chain is documented
- VM1 (ELK + Fleet) is running — Kibana should be open so you can observe telemetry arriving as the attack unfolds in later steps

---

## VM setup (VMware)

### Specs
| Setting | Value |
|---|---|
| Name | `mythicc2` |
| OS | Ubuntu 22.04 LTS Server (64-bit, no GUI) |
| RAM | 2048 MB |
| CPU | 2 cores |
| Disk | 40 GB (dynamically allocated) |
| Adapter 1 | Host-Only (`VMnet2` — lab network) |
| Adapter 2 | NAT (internet — needed for apt and git clone) |

> Unlike the target VMs, Mythic C2 **needs internet access** during setup to pull Docker images and clone the repository. Keep the NAT adapter enabled during installation, then you can disable it after setup is complete if you want to isolate the attacker VM.


---

## Add a swapfile

Mythic runs multiple Docker containers simultaneously. On a 2 GB VM this will cause OOM kills without swap:

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Make permanent across reboots
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Verify
free -h
# Swap line should show 2.0G
```

---

## Step 1 — Update the system

```bash
sudo apt-get update && sudo apt-get upgrade -y
```

---

## Step 2 — Install prerequisites

Mythic requires Docker, Docker Compose, and Make:

```bash
sudo apt install -y docker-compose make
```

Verify installations:
```bash
docker --version
# Docker version 24.x.x or similar

docker-compose --version
# docker-compose version 1.29.x or similar

make --version
# GNU Make 4.x
```

---

## Step 3 — Clone the Mythic repository

```bash
cd ~
git clone https://github.com/its-a-feature/Mythic
cd Mythic
```

Confirm the directory structure:
```bash
ls
# Should show: docker-compose.yml, Makefile, mythic-cli, .env, etc.
```

---

## Step 4 — Start Mythic

```bash
sudo make
```

This command:
1. Builds and pulls all required Docker containers
2. Generates a random admin password and stores it in `.env`
3. Starts all Mythic services (server, database, rabbitmq, nginx)

> This will take **5–10 minutes** on first run — Docker needs to pull several container images. 

Monitor the output — you should see containers starting one by one. When
complete, the terminal returns to the prompt.

---

## Step 5 — Retrieve the admin credentials

Mythic auto-generates credentials on first start and stores them in `.env`:

```bash
sudo cat .env | grep -E "MYTHIC_ADMIN|MYTHIC_PASS"
```

Expected output:
```
MYTHIC_ADMIN_USER=mythic_admin
MYTHIC_ADMIN_PASSWORD=<randomly_generated_password>
```

---

## Step 6 — Check all containers are running

```bash
sudo docker ps
```

You should see containers running for all Mythic services:

| Container | Purpose |
|---|---|
| mythic_server | Core Mythic application server |
| mythic_postgres | PostgreSQL database |
| mythic_rabbitmq | Message queue between services |
| mythic_nginx | Reverse proxy serving the web UI |
| mythic_documentation | Embedded docs (optional) |

All containers should show status `Up` with no restarts.

---

## Step 7 — Allow Mythic UI port through UFW

```bash
sudo ufw allow from 172.31.0.0/24 to any port 7443
sudo ufw allow OpenSSH
sudo ufw enable
sudo ufw status
```

---

## Step 8 — Access the Mythic web UI

From your **Windows host browser**, navigate to:
```
https://172.31.0.134:7443
```

Accept the self-signed certificate warning.

Login with:
- **Username:** `mythic_admin`
- **Password:** (from `.env` file above)

You should see the Mythic dashboard — an empty callbacks table and a sidebar with icons for Payloads, C2 Profiles, Operations, and more.

---

## Step 9 — Verify Mythic CLI is working

The `mythic-cli` binary is used to install agents and C2 profiles in the next step:

```bash
sudo ./mythic-cli --help
# Should show: available mythic-cli commands
```

---

## Take a VMware snapshot

**Before proceeding to next step, take a snapshot of VM4 right now.**

---


## Key takeaway
Mythic is a modern, modular C2 framework used by both red teams and threat actors. Its architecture separates the **server**, **agents**, and **C2 profiles** into independently installable components — meaning a real adversary can swap out any layer without changing the others. Understanding this modularity is directly relevant to detection: the Sysmon telemetry you will hunt in focuses on **agent behaviour** (process creation, network connections), not on Mythic itself, because Mythic's server is never visible on the target endpoint — only the agent it generates is.

---


## VM status at end of step

| VM | Status |
|---|---|
| ELK + Fleet (VM1) | ✅ Running — Elasticsearch :9200, Kibana :5601, Fleet :8220 |
| Windows target (VM2) | ⏸ Suspended — not needed  |
| Ubuntu target (VM3) | ⏸ Suspended — not needed until further notice |
| Mythic C2 (VM4) | ✅ Running — Mythic UI accessible on :7443 |
| osTicket (host) | Not yet installed |