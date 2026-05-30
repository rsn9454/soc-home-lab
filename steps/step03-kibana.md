# Step 03 — Install Kibana (VM1)

## Objective
Install Kibana on the same Ubuntu VM as Elasticsearch, connect it to the local Elasticsearch instance, and access the Kibana UI from the Windows host browser on port 5601.

---

## Prerequisites
- Step 02 complete — Elasticsearch is running and reachable on `:9200`
- You have the `elastic` superuser password noted from the Step 02 install output
- You have the Elasticsearch enrollment token (also from Step 02 install output)

> If you lost the enrollment token, regenerate it:
> ```bash
> sudo /usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana
> ```

---

## Installation

### 1. Download Kibana package

```bash
wget https://artifacts.elastic.co/downloads/kibana/kibana-9.4.2-amd64.deb
```
### 1. Install the package

```bash
dpkg -i kibana-9.4.2-amd64.deb
```

---

## Configuration

### Edit `kibana.yml`
```bash
sudo nano /etc/kibana/kibana.yml
```

Key changes to make:

```yaml
# ── Server ───────────────────────────────────────────────
# Allow access from the Windows host browser
# Replace with your host-only IP
server.port: 5601
server.host: 172.31.0.128
```

> Leave all other settings at their defaults for this lab.
---

## Enroll Kibana with Elasticsearch

Kibana uses an enrollment token to securely pair itself with Elasticsearch. This configures the TLS certificates and service account token automatically.

### Create the enrollment token
```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana
```

---

## Firewall rules

Allow port 5601 from the host-only subnet so the Windows host browser can reach Kibana:

```bash
sudo ufw allow from 172.31.0.0/24 to any port 5601
sudo ufw status
```

---

## Start and enable the service

```bash
sudo systemctl daemon-reload
sudo systemctl enable kibana
sudo systemctl start kibana
```

Check status:
```bash
sudo systemctl status kibana
# Should show: active (running)
```

Kibana takes **60–90 seconds** to fully start — it runs optimisations on first launch. Check the log if the browser shows nothing:

```bash
sudo journalctl -u kibana -f
# Wait for: "Kibana is now available"
```

---

## First login from Windows host

Open a browser on your Windows host and go to:

```
http://172.31.0.128:5601
```

Login with:
- **Username:** `elastic`
- **Password:** `YOUR_ELASTIC_PASSWORD` (from Step 02)

On first login Kibana may ask for a **verification code** — run this on the VM to get it:

```bash
sudo /usr/share/kibana/bin/kibana-verification-code
```

Enter the 6-digit code in the browser. You will only be asked once.


---

## Configs saved to repo

Saved under `configs/kibana/` with sensitive values redacted:

**`configs/kibana/kibana.yml`**

---


## Key takeaway

Kibana is the analyst's primary interface in this lab — every detection rule, dashboard, alert investigation, and Fleet management task happens here. The enrollment token flow in Kibana is a significant improvement over manually configuring certificates: one command handles TLS trust, authentication, and service account creation in one step. In production, this token-based enrollment extends to Fleet Server and Elastic Agents as well, which is exactly what the next step covers.

---

## VM status at end of step

| VM | Status |
|---|---|
| ELK + Fleet (VM1) | ✅ Running — Elasticsearch :9200 + Kibana :5601 |
| Windows target (VM2) | Not yet created |
| Ubuntu target (VM3) | Not yet created |
| Mythic C2 (VM4) | Not yet created |
| osTicket (host) | Not yet installed |