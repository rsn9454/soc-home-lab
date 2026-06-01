# Step 06 — Install and Configure Sysmon (VM2)

## Objective
Install Sysmon (System Monitor) on the Windows Server 2022 target VM using the Olaf configuration, verify telemetry appears in Windows Event Viewer, and add the Sysmon integration to the Fleet `windows-endpoints` policy so Sysmon events flow into Elasticsearch and are queryable in Kibana.

---

## Prerequisites
- Step 05 complete — Windows Server 2022 VM2 is enrolled in Fleet with the `windows-endpoints` policy
- Elastic Agent on VM2 is showing **Healthy** in Kibana Fleet UI
- VM2 is booted and you are logged in as Administrator

---

## What is Sysmon and why does it matter?

Windows built-in event logging captures authentication events (4624, 4625), process creation (4688), and service changes — but the detail is limited. Sysmon extends this dramatically:

| Sysmon Event ID | What it captures | Why it matters for SOC |
|---|---|---|
| 1 | Process creation with full command line + parent | Detect malware execution, LOLBins, C2 agents |
| 3 | Network connection (process → IP:port) | Detect C2 beacons, lateral movement |
| 7 | Image (DLL) loaded | Detect DLL hijacking |
| 8 | CreateRemoteThread | Detect process injection |
| 11 | File created | Detect dropped payloads |
| 12/13 | Registry key created/set | Detect persistence mechanisms |
| 22 | DNS query | Detect C2 domain lookups |

Without Sysmon, the Mythic C2 simulation in later steps would generate almost no useful telemetry. With Sysmon, every process the Apollo agent spawns, every network connection it makes, and every file it drops is logged and searchable in Kibana.

---

## Step 1 — Download Sysmon

Go to https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon
Download the zip file
Extract the contents

> Always use **Sysmon64.exe** on a 64-bit OS (Windows Server 2022 is 64-bit).

---

## Step 2 — Download the Olaf config

Go to https://github.com/olafhartong/sysmon-modular/blob/master/sysmonconfig.xml
Download the .xml file
Place it inside the Sysmon folder

> If the VM has no internet access, download it on your Windows host and transfer via a VMware shared folder.

---

## Step 3 — Install Sysmon with the config

```powershell
cd C:\<path-to>\Sysmon

.\Sysmon64.exe -accepteula -i sysmonconfig.xml
```

Expected output:
```
System Monitor v15.x - System activity monitor
Copyright (C) 2014-2024 Mark Russinovich and Thomas Garnier
Sysinternals - www.sysinternals.com

Loading configuration file with schema version 4.x
Sysmon schema version: 4.x
Configuration file validated.
Sysmon64 installed.
SysmonDrv installed.
Starting SysmonDrv.
SysmonDrv started.
Starting Sysmon64.
Sysmon64 started.
```

---

## Step 4 — Verify Sysmon is running

### Check the service
```powershell
Get-Service -Name "Sysmon64"
# Status should be: Running
```

### Check Event Viewer
1. Open **Event Viewer** (run `eventvwr.msc`)
2. Navigate to:
   ```
   Applications and Services Logs
   └── Microsoft
       └── Windows
           └── Sysmon
               └── Operational
   ```
3. You should see events already populating — particularly Event ID 1 (process creation) from your own PowerShell session

---

## Step 5 — Add the Sysmon integration in Kibana Fleet

Now that Sysmon is installed and generating events, add the integration to the `windows-endpoints` policy so the Elastic Agent forwards Sysmon logs to Elasticsearch.

1. In Kibana, go to **Management → Fleet → Agent policies**
2. Click **windows-endpoints**
3. Click **Add integration**
4. Search for **Custom Windows Event Logs**
5. Under Configure integration → Integration name → Type **Microsoft-Windows-Sysmon/Operational**
6. Click **Add integration**

The Elastic Agent on VM2 will receive the updated policy within 30 seconds and begin forwarding Sysmon events.

---

## Step 6 — Verify Sysmon logs in Kibana

1. Go to **Analytics → Discover** in Kibana
2. Filter for Sysmon events:

```kql
agent.name: "WIN-KT9TVSF1T0C" AND
event.provider: "Microsoft-Windows-Sysmon"
```

You should see events flowing in. Drill down into an Event ID 1 entry and note the fields.



Added these to `resources/kql-cheatsheet.md` — you will use them heavily in later steps.

---

## Useful Sysmon KQL queries (added to cheatsheet)

```kql
# All Sysmon process creation events from Windows target
winlog.event_id: "1" AND event.provider: "Microsoft-Windows-Sysmon"

# Sysmon network connections from Windows target
winlog.event_id: "3" AND event.provider: "Microsoft-Windows-Sysmon"

# Sysmon events by a specific process name
winlog.event_id: "1" AND winlog.event_data.Image: "*powershell.exe"

# Sysmon DNS queries (Event ID 22)
winlog.event_id: "22" AND event.provider: "Microsoft-Windows-Sysmon"

# Sysmon file creation events
winlog.event_id: "11" AND event.provider: "Microsoft-Windows-Sysmon"
```

---

## Configs saved to repo

**`configs/sysmon/sysmonconfig.xml`**

Saved a copy of the Olaf config file to the repo. This is one of the most valuable files in the repo — it documents exactly what telemetry is being collected.


---



## Key takeaway

Sysmon is the single biggest upgrade to Windows endpoint visibility available at zero cost. The difference between raw Windows event logs and Sysmon-enriched logs is the difference between seeing "a process started" and seeing "PowerShell spawned by SYSTEM, ran encoded command, connected to 194.x.x.x on port 443 three seconds later." In later steps, every stage of the Mythic C2 attack chain will be visible in Kibana precisely because Sysmon is capturing process creation, network connections, and file drops. Without it, most of that activity would be invisible.

---

## VM status at end of step

| VM | Status |
|---|---|
| ELK + Fleet (VM1) | ✅ Running — Elasticsearch :9200, Kibana :5601, Fleet :8220 |
| Windows target (VM2) | ✅ Running — Elastic Agent healthy, Sysmon installed, all logs flowing |
| Ubuntu target (VM3) | Not yet created |
| Mythic C2 (VM4) | Not yet created |
| osTicket (host) | Not yet installed |