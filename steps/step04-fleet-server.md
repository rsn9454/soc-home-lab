# Step 04 — Set up Fleet Server (VM1)

## Objective
Install and enroll an Elastic Agent on VM1 in **Fleet Server mode**, making it the central management point for all Elastic Agents deployed across the lab. After this step, every new endpoint (Windows target, Ubuntu target) can be enrolled from the Kibana Fleet UI with a single command.

---

## Prerequisites
- Step 03 complete — Kibana is running and accessible at `http://172.31.0.128:5601`
- You are logged into Kibana as `elastic`
- VM1 has both adapters working (NAT for internet, host-only for lab network)

---

## Concept: what Fleet Server actually is

```
┌─────────────────────────────────────────────────────┐
│                   Kibana UI                         │
│         (you manage everything from here)           │
└──────────────────────┬──────────────────────────────┘
                       │ Policy + config
                       ▼
              ┌────────────────┐
              │  Fleet Server  │  ← this step
              │   :8220        │
              └───────┬────────┘
          ┌───────────┼───────────┐
          ▼           ▼           ▼
    [Windows VM]  [Ubuntu VM]  [future agents]
   Elastic Agent  Elastic Agent
```

Fleet Server sits between Kibana and all enrolled agents. It distributes policies, receives check-ins, and relays data. It runs as an Elastic Agent itself — just in a special server role. Co-locating it on VM1 saves an entire VM, which matters on 8 GB.

---

## Step 1 — Generate a Fleet Server service token (Kibana UI)

1. In Kibana, go to **Management → Fleet**
2. On first visit, Fleet will prompt you with a setup wizard — click **Get started**
3. Under **Fleet Server hosts**, set the URL to:
   ```
   https://172.31.0.128:8220
   ```
4. Click **Generate Fleet Server policy** — Kibana creates a default Fleet Server agent policy
5. On the next screen, select **Quick start** (self-signed certificates — fine for a lab)
6. Kibana will display a `elastic-agent install` command with an embedded **service token** — copy the entire command


---

## Step 2 — Install Fleet Server to a centralized host

Run the entire command in VM1

---

## Step 3 — Verify Fleet Server is running

### Check the service
```bash
sudo systemctl status elastic-agent
# Should show: active (running)
```

### Check logs
```bash
sudo journalctl -u elastic-agent -f
# Look for: "Fleet Server - Ready to serve connections"
```

### Verify in Kibana
1. Go to **Management → Fleet → Agents**
2. You should see one agent listed with:
   - **Status:** Healthy
   - **Policy:** Fleet Server policy
   - **Version:** matching your Elasticsearch version

---

## Firewall rules

Allow port 8220 from the host-only subnet so future agents on VM2 and VM3 can check in:

```bash
sudo ufw allow from 172.31.0.0/24 to any port 8220
sudo ufw status
```

---

## Step 4 — Create agent policies for endpoints (Kibana UI)

Before enrolling VM2 and VM3, create dedicated policies for them in Fleet. This keeps their configurations separate from the Fleet Server policy.

### Create a Windows endpoint policy
1. Go to **Management → Fleet → Agent policies**
2. Click **Create agent policy**
3. Name: `windows-endpoints`
4. Leave other settings at default → **Create agent policy**

### Create a Linux endpoint policy
1. Repeat the above
2. Name: `linux-endpoints`
3. **Create agent policy**

You will add integrations (Windows Event Logs, System metrics, Sysmon) to these policies once the agents are enrolled.

---

## Configs saved to repo

Saved under `configs/fleet/`:

**`configs/fleet/fleet-server-notes.md`**
```markdown
## Fleet Server configuration notes

- Host URL: https://REDACTED:8220
- Policy name: Fleet Server Policy
- Elasticsearch host: https://REDACTED:9200
- Service: elastic-agent (systemd)

## Agent policies created
- windows-endpoints (for VM2)
- linux-endpoints (for VM3)
```

---

## Key takeaway

Fleet Server is the operational backbone of the Elastic Agent ecosystem — without it, agents can't receive policy updates, integrations can't be pushed, and the SIEM has no centralised agent management. Co-locating Fleet Server on the same VM as ELK is a deliberate lab compromise: in production, Fleet Server typically runs on a dedicated host or cluster to handle thousands of agent check-ins. 

---

## VM status at end of step

| VM | Status |
|---|---|
| ELK + Fleet (VM1) | ✅ Running — Elasticsearch :9200, Kibana :5601, Fleet Server :8220 |
| Windows target (VM2) | Not yet created |
| Ubuntu target (VM3) | Not yet created |
| Mythic C2 (VM4) | Not yet created |
| osTicket (host) | Not yet installed |