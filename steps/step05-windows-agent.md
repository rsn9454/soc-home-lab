# Step 05 — Enroll Windows Server 2022 (VM2)

## Objective
Create a Windows Server 2022 VMware VM, configure it on the host-only network, install Elastic Agent via Fleet enrollment, and confirm the agent appears as healthy in the Kibana Fleet UI — making Windows event log telemetry available in Elasticsearch.

---

## Prerequisites
- Step 04 complete — Fleet Server is running and healthy on VM1 at `https://172.31.0.128:8220`
- The `windows-endpoints` agent policy exists in Kibana Fleet
- VM1 (ELK + Fleet) is running
- You have enough free RAM — **suspend any other VMs before booting VM2** (Windows Server 2022 needs 2 GB)

---

## VM setup (VMware)

### Specs
| Setting | Value |
|---|---|
| Name | `Windows Server 2022` |
| OS | Windows Server 2022 (64-bit) |
| RAM | 2048 MB |
| CPU | 2 cores |
| Disk | 50 GB (dynamically allocated) |
| Adapter 1 | Host-Only (`VMnet2` — lab network) |

> Only one adapter needed here — no NAT required. All downloads and agent enrollment happen from within the host-only network. Internet access for this VM is intentionally withheld to simulate a protected endpoint.


### Windows Server 2022 installation notes
- Select **Windows Server 2022 Standard Evaluation (Desktop Experience)** during install — the GUI version makes it easier to copy/paste enrollment commands
- Skip product key when prompted (evaluation license is fine for the lab)

---

## Network configuration (inside Windows VM)

After install, configure a static IP on the host-only adapter so it stays consistent across reboots.

1. Open **Network and Sharing Center → Change adapter settings**
2. Right-click the Ethernet adapter → **Properties**
3. Select **Internet Protocol Version 4 (TCP/IPv4) → Properties**
4. Set:

```
IP address:      172.31.0.20
Subnet mask:     255.255.255.0
Default gateway: 172.31.0.1
DNS server:      8.8.8.8
```

---

## Get the enrollment command from Kibana

1. In Kibana, go to **Management → Fleet → Agent policies**
2. Click on **windows-endpoints**
3. Click **Add agent** (top right)
4. Select **Enroll in Fleet**
5. Under **What type of host are you enrolling?** → select **Windows**
6. Kibana generates a full PowerShell command — copy it

It will look like this:
```powershell
$ProgressPreference = 'SilentlyContinue'
Invoke-WebRequest -Uri https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-8.x.x-windows-x86_64.zip -OutFile elastic-agent-8.x.x-windows-x86_64.zip
Expand-Archive .\elastic-agent-8.x.x-windows-x86_64.zip -DestinationPath .
cd elastic-agent-8.x.x-windows-x86_64
.\elastic-agent.exe install --url=https://192.168.56.10:8220 --enrollment-token=YOUR_TOKEN --insecure
```

---

## Install Elastic Agent on VM2

### Run directly from Kibana's generated command
Open PowerShell **as Administrator** on VM2 and paste the full command copied from Kibana.


### Expected output
```
Elastic Agent has been successfully installed.
The agent is running and enrolled to Fleet Server.
```

The agent installs as a Windows service named **Elastic Agent** and starts automatically.

---

## Verify enrollment

### In Kibana Fleet UI
1. Go to **Management → Fleet → Agents**
2. You should now see two agents:
   - **VM1** — Fleet Server policy — Healthy
   - **VM2** — windows-endpoints policy — Healthy
3. Click on the VM2 agent entry and confirm:
   - Operating system: Windows
   - Agent version: matches Elasticsearch version
   - Last check-in: within the last 30 seconds

---

## Verify logs arriving in Kibana

1. Go to **Analytics → Discover**
2. You should see Windows Security, System, and Application event logs flowing in within 60 seconds of deploying the integrations

A quick sanity check — filter for a known event:
```kql
agent.name: "WIN-KT9TVSF1T0C" AND event.code: "4624"
# Event 4624 = successful logon — should appear every time you log into the VM
```

Added this query to `resources/kql-cheatsheet.md`.

---

## Key takeaway

Enrolling an endpoint via Fleet rather than configuring Beats individually is the modern Elastic approach — one policy change in Kibana propagates to every enrolled agent automatically. In a real SOC environment with hundreds of endpoints, this centralised management is the difference between a maintainable detection platform and a configuration nightmare. The `windows-endpoints` policy created here is the foundation that all Windows-specific integrations (Sysmon, Elastic Defend) will build on.

---

## VM status at end of step

| VM | Status |
|---|---|
| ELK + Fleet (VM1) | ✅ Running — Elasticsearch :9200, Kibana :5601, Fleet :8220 |
| Windows target (VM2) | ✅ Running — Elastic Agent enrolled, Windows logs flowing |
| Ubuntu target (VM3) | Not yet created |
| Mythic C2 (VM4) | Not yet created |
| osTicket (host) | Not yet installed |