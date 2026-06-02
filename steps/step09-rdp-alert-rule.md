# Step 09 — Create RDP Brute Force Alert Rule

## Objective
Build a threshold-based detection rule in Kibana Security that fires when more than 5 failed RDP login attempts are detected from the same source IP within 2 minutes against the Windows Server target — mirroring the SSH brute force rule from previous step, but targeting Windows authentication telemetry via Windows Event Log event codes.

---

## Prerequisites
- RDP brute force traffic has been generated against VM2 and event code `4625` is visible in Kibana Discover
- Elastic Agent on VM2 is healthy and forwarding Windows Security event logs
- The `windows-endpoints` Fleet policy has the Windows integration enabled

---

## Step 1 — Confirm RDP telemetry in Discover

Before building the rule, verify the data exists and the fields are correct.

Go to **Analytics → Discover** and run:
```kql
agent.name: "WIN-KT9TVSF1T0C" AND
event.code: "4625"
```

Add these columns to the Discover table to inspect the data:

| Field | What it shows |
|---|---|
| `event.code` | Should be `4625` for all results |
| `winlog.event_data.TargetUserName` | The username the attacker tried |
| `source.ip` | Source IP of the RDP attempt |


If the query returns results, the data is confirmed and you can proceed. If it returns 0 results, boot VM2, generate RDP failures from your host (attempted RDP logins with wrong passwords), wait 60 seconds, then recheck.

---

## Step 2 — Navigate to Detection Rules

In Kibana go to:
```
Security → Rules → Detection rules (SIEM) → Create new rule
```

---

## Step 3 — Select rule type

Select **Threshold** and click **Next**.

---

## Step 4 — Define the rule query

### Index pattern
```
logs-*
```

### Custom query
```kql
agent.name: "WIN-KT9TVSF1T0C" AND
event.code: "4625"
```

### Threshold settings

| Setting | Value | Reason |
|---|---|---|
| Group by | `source.ip` | Evaluate each attacker IP independently |
| Threshold count | `5` | More than 5 failures triggers the alert |


Click **Preview alerts** to verify the query matches historical brute force data.

Click **Next**.

---

## Step 5 — Configure rule details

| Field | Value |
|---|---|
| Name | `RDP Brute Force Activity - WIN-KT9TVSF1T0C` |
| Description | `Detects if the same IP address fails to login 5+ times in the last 2 minutes to the windows agent 'WIN-KT9TVSF1T0C'` |
| Severity | **Low** |
| Risk score | `21` |
| Tags | `brute-force`, `rdp`, `windows`, `T1110.001` |

### MITRE ATT&CK mapping
Click **Add MITRE ATT&CK threat**:

| Field | Value |
|---|---|
| Tactic | Credential Access |
| Technique | T1110 — Brute Force |
| Sub-technique | T1110.001 — Password Guessing |

Click **Next**.

---

## Step 6 — Set the rule schedule

| Setting | Value |
|---|---|
| Runs every | `2 minute` |
| Additional look-back time | `0 minutes` |

Click **Next**.

---

## Step 7 — Rule actions (skip for now)

Leave the Actions section empty — osTicket integration is added in later step.

Click **Create & enable rule**.

---

## Step 8 — Verify the rule is active

After saving, confirm on the rule detail page:

- **Status:** Active (green)
- **Last execution:** shows a timestamp within the last minute
- **Last execution status:** succeeded

---

## Step 9 — Trigger and confirm the alert fires

### Generate RDP brute force traffic from your Windows host

**Manual RDP attempts**
Open Remote Desktop Connection (`mstsc`) on your Windows host, connect to `172.131.0.20`, and enter wrong passwords at least 6 times in quick succession.

Wait 1–2 minutes for the rule engine to run.

### Check Security → Alerts
Go to **Security → Alerts**. You should see:
- **Rule name:** RDP Brute Force Activity - WIN-KT9TVSF1T0C
- **Severity:** Low
- **Source:** `source.ip` showing your host IP

Click into the alert and compare the fields to the SSH brute force alert from the previous step. Note the structural differences — Windows event fields (`winlog.event_data.*`) versus Linux auth fields (`system.auth.ssh.*`).

---

## Compare the two rules side by side

Now that both rules exist, it is worth reviewing them together. Go to
**Security → Rules → Detection rules (SIEM)** and confirm both are active:

| Rule | Target | Event source | Threshold field |
|---|---|---|---|
| SSH Brute Force - Failed Attempts - Ubuntu Target | VM3 | `system.auth.ssh.event` | `source.ip` |
| RDP Brute Force - Failed Attempts - Windows Target | VM2 | `event.code: 4625` | `source.ip` |

This side-by-side view is a clear illustration of cross-platform SOC coverage — the same attack pattern (credential brute force) detected across two different operating systems using completely different telemetry sources.

---

## Export the rule for the repo

1. Go to **Security → Rules → Detection rules (SIEM)**
2. Find `RDP Brute Force Activity - WIN-KT9TVSF1T0C`
3. Click the three dots **(⋮)** → **Export rule**
4. Save the downloaded file as:
   ```
   detection-rules/rdp-brute-force-rule.json
   ```

---

## KQL queries used (add to cheatsheet)

```kql
# All failed logon events on Windows target
agent.name: "WIN-KT9TVSF1T0C" AND event.code: "4625"

# RDP failed logons only (LogonType 10) (if you are using local network then LogonType is 3)
agent.name: "WIN-KT9TVSF1T0C" AND
event.code: "4625" AND
winlog.event_data.LogonType: "10"

# RDP failed logons from a specific IP
agent.name: "WIN-KT9TVSF1T0C" AND
event.code: "4625" AND
winlog.event_data.LogonType: "10" AND
source.ip: "172.31.0.1"

# RDP successful logins (for comparison / baselining)
agent.name: "WIN-KT9TVSF1T0C" AND
event.code: "4624" AND
winlog.event_data.LogonType: "10"

# All failed logons by username (brute force username enumeration)
agent.name: "WIN-KT9TVSF1T0C" AND
event.code: "4625" AND
winlog.event_data.LogonType: "10"
# Add winlog.event_data.TargetUserName as a column in Discover
```

---

## Key takeaway
RDP is one of the most attacked services on the internet — exposed RDP ports are scanned and brute forced constantly, which is why LogonType filtering matters so much here. A rule that fires on all `4625` events without the LogonType filter would generate hundreds of false positives in any real environment (failed local logons, network share access failures, service account password expiries). The discipline of building precise queries — and understanding *why* each filter exists — is what makes a detection engineer's rules trustworthy enough to act on. In later steps, both this rule and the SSH rule will be reviewed for further tuning.

---

## VM status at end of step

| VM | Status |
|---|---|
| ELK + Fleet (VM1) | ✅ Running — Elasticsearch :9200, Kibana :5601, Fleet :8220 |
| Windows target (VM2) | ⏸ Suspended — boot only to generate test traffic |
| Ubuntu target (VM3) | ⏸ Suspended — not needed for this step |
| Mythic C2 (VM4) | Not yet created |
| osTicket (host) | Not yet installed |