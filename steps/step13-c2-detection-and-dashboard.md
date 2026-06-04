# Step 13 — Apollo C2 Detection Rule + Mythic C2 Activity Dashboard

## Objective
Build a Kibana detection alert rule that identifies the Apollo C2 agent on the Windows target using Sysmon process creation telemetry, then construct a three-panel dashboard that gives a unified view of C2 process activity, outbound network connections, and defense evasion — turning the raw Sysmon logs into actionable detections.

---

## Prerequisites
- Apollo payload was executed on VM2 and the C2 callback was confirmed in Mythic
- Sysmon Event ID 1 and Event ID 3 entries for the Apollo payload are visible in Kibana Discover


---

## Background — why these three event IDs

The dashboard is built around three Sysmon/Defender event IDs that togethercover the key stages of C2 activity observed in Step 22:

| Event ID | Source | What it captures | Attack stage |
|---|---|---|---|
| 1 | Sysmon | Process creation — every new process with full command line | Apollo agent execution |
| 3 | Sysmon | Network connection initiated by a process | C2 beacon to Mythic |
| 5001 | Windows Defender | Real-time protection disabled | Defense evasion |

---

## Part 1 — Build the Apollo detection alert rule

### Step 1 — Find the Apollo payload in Discover

1. In the KQL bar, search for your payload filename:
   ```kql
   apollo.exe
   ```
2. Change the sort order from **Newest** to **Oldest** — this shows the first moment the payload appeared on the system

### Step 2 — Inspect the process creation event

Click into the first Sysmon Event ID 1 entry for the payload and note these two fields — they are the basis of the detection rule:

| Field | Value | Notes |
|---|---|---|
| `winlog.event_data.OriginalFileName` | `Apollo.exe` | The compiled name baked into the binary — survives file renaming |
| `winlog.event_data.Hashes` | `SHA256=<your hash>` | Unique fingerprint of your specific payload build |


> **Why use `OriginalFileName` and hash rather than `Image`?** An attacker can rename the payload file to anything — `svchost.exe`, `update.exe`, etc. The `OriginalFileName` field is embedded in the PE header at compile time and does not change when the file is renamed. The SHA256 hash is even more specific — it uniquely identifies this exact payload build. Both together make a highly precise detection.

### Step 3 — Refine the query

Update the KQL bar to:
```kql
event.code: "1" AND
winlog.event_data.OriginalFileName: Apollo.exe)
```

Confirm the payload execution event appears in the results.

### Step 4 — Navigate to Detection Rules

Go to **Security → Rules → Detection rules (SIEM) → Create new rule**

### Step 5 — Select rule type

Select **Custom query** and click **Next**.

> A custom query rule (not threshold) is correct here — any single execution of the Apollo agent should immediately fire the alert. There is no legitimate reason for `OriginalFileName: Apollo.exe` to appear on an endpoint.

### Step 6 — Define the rule query

**Index pattern:**
```
logs-*
```

**Custom query:**
```kql
event.code: "1" AND
(winlog.event_data.OriginalFileName: "Apollo.exe" OR
winlog.event_data.Hashes: *YOUR_PAYLOAD_SHA256_HERE*)
```

> Replace `YOUR_PAYLOAD_SHA256_HERE` with the actual hash you noted from the Event ID 1 log in Step 2. Keep the `*` wildcards on both sides — the Hashes field contains the full string `SHA256=abc123...` so partial matching is needed.

Click **Preview alerts** — the Apollo execution should appear as a match.

Click **Next**.

### Step 7 — Configure rule details

| Field | Value |
|---|---|
| Name | `Mythic C2 Apollo Agent Detected` |
| Description | `This rule detects a potential mythic c2 apollo agent.` |
| Severity | **Critical** |
| Risk score | `99` |
| Tags | `c2`, `mythic`, `apollo`, `T1071.001`, `T1204.002` |

### Step 8 — MITRE ATT&CK mapping

| Tactic | Technique | ID |
|---|---|---|
| Execution | User Execution: Malicious File | T1204.002 |
| Command & Control | Application Layer Protocol: Web Protocols | T1071.001 |

Click **Next**.

### Step 9 — Set the rule schedule

| Setting | Value |
|---|---|
| Runs every | `5 minutes` |
| Additional look-back time | `5 minute` |

Click **Next** → leave Actions empty → **Create & enable rule**.

### Step 10 — Verify the rule fires

Confirm under **Security → Rules** that the rule shows **Active** and its last execution status is **succeeded**.

Go to **Security → Alerts** — the Apollo execution should appear as a **Critical** severity alert.

---

## Part 2 — Build the Mythic C2 Activity Dashboard

The dashboard has three panels, each built from a saved search. Build them one at a time.

### Navigate to Dashboards
Go to **Analytics → Dashboards → Create dashboard**

---

### Panel 1 — C2 Process Creation (Sysmon Event ID 1)

**Purpose:** Show all process creation events on the Windows target — a table view that surfaces any suspicious process names and command lines.

1. Click **Create visualization**
2. Select visualization type: **Table**
3. In the KQL bar:
   ```kql
   event.code: 1 AND event.provider: Microsoft-Windows-Sysmon AND (powershell or cmd or rundll32)
   ```
4. Configure the table:
   - **Add relevant fields**
5. Title: `Process Created (PowerShell, CMD, Rundl32)`
6. Click **Save and return**

---

### Panel 2 — Outbound Network Connections (Sysmon Event ID 3)

**Purpose:** Show all outbound network connections initiated by processes — the C2 beacon from Apollo to Mythic will appear here.

1. Click **Create visualization**
2. Select visualization type: **Table**
3. In the KQL bar:
   ```kql
   event.code: 3 AND event.provider: Microsoft-Windows-Sysmon AND winlog.event_data.Initiated: true 

   ```
4. Configure the table:
   - **Add relevant fields**
5. Title: `Process Initiated Network Connections`
6. Click **Save and return**

---

### Panel 3 — Windows Defender Disabled (Event ID 5001)

**Purpose:** Alert-style panel showing any instances of Windows Defender real-time protection being turned off — the defense evasion step.

1. Click **Create visualization**
2. Select visualization type: **Table**
3. In the KQL bar:
   ```kql
   event.code: 5001 AND event.provider: Microsoft-Windows-Windows Defender
   ```
4. Configure the table - add relevant fields.
5. Title: `Microsoft Defender Disabled`
6. Click **Save and return**

---

### Save the dashboard

Click **Save** → name it:
```
Mythic C2 Activity Dashboard
```

---

## Export artifacts for the repo

### Export the detection rule
1. Go to **Security → Rules → Detection rules (SIEM)**
2. Find `Mythic C2 Apollo Agent Detected`
3. Click **(⋮) → Export rule**
4. Save as:
   ```
   detection-rules/mythic-c2-apollo-process-creation.json
   ```

### Export the dashboard
1. Go to **Management → Stack Management → Saved Objects**
2. Search for `Mythic C2 Activity Dashboard`
3. Tick → **Export**
4. Save as:
   ```
   dashboards/mythic-c2-activity-dashboard.ndjson
   ```

---

## KQL queries used (add to cheatsheet)

```kql
# Process Creation (PowerShell, CMD, Rundll32)
event.code: 1 AND event.provider: Microsoft-Windows-Sysmon AND (powershell or cmd or rundll32)

# Process Initiated Network Connections
event.code: 3 AND event.provider: Microsoft-Windows-Sysmon AND winlog.event_data.Initiated: true 

# Microsoft Defender Disabled
event.code: 5001 AND event.provider: Microsoft-Windows-Windows Defender

#Apollo agent detection
event.code: 1 AND (winlog.event_data.Hashes: *6FC88C6EE1298FA03ADE7125E2E040C3946C8535ACCB455EB16225C5EE55A3DD* OR winlog.event_data.OriginalFileName: Apollo.exe)

```

---

## Key takeaway
The most important insight in this step is the distinction between detecting by **filename** versus detecting by **PE metadata**. Renaming `Apollo.exe` to `svchost.exe` defeats a filename-based rule entirely — but `OriginalFileName` and the SHA256 hash are embedded in the binary itself and cannot be changed without recompiling the payload. This is why detection engineers layer their rules: filename for quick wins, `OriginalFileName` for resilience against renaming, and hash for the highest-confidence match. In a real SOC, a High severity alert on a known C2 agent hash would trigger an immediate response — no investigation needed to confirm severity.

---

## VM status at end of step

| VM | Status |
|---|---|
| ELK + Fleet (VM1) | ✅ Running — Elasticsearch :9200, Kibana :5601, Fleet :8220 |
| Windows target (VM2) | ⏸ Suspended — telemetry already in Elasticsearch |
| Ubuntu target (VM3) | ⏸ Suspended |
| Mythic C2 (VM4) | ⏸ Suspended — not needed until investigation write-up |
| osTicket (host) | Not yet installed |