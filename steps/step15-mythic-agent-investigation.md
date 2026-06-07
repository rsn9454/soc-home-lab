# Step 15 — Investigate Mythic C2 Apollo Agent Alert

## Objective
Work through the Critical severity `Mythic C2 Apollo Agent Detected` alert in Kibana Security using a structured investigation methodology — pivoting from the initial alert through Sysmon telemetry to reconstruct the full attack chain, confirm the C2 beacon, document post-access activity, and produce the formal investigation write-up.

---

## Prerequisites
- Apollo agent was executed on VM2 and C2 callback confirmed
- Apollo detection rule is active and has fired alerts
- osTicket is connected and a ticket was auto-generated
- VM1 (ELK + Fleet) is running — Kibana accessible

---

## Investigation framework for C2 alerts

| # | Question | Data source | Key fields |
|---|---|---|---|
| 1 | What process triggered the alert? | Sysmon Event ID 1 | Image, OriginalFileName, CommandLine, ParentImage, Hashes |
| 2 | Did it establish a network connection? | Sysmon Event ID 3 | DestinationIp, DestinationPort, Initiated |
| 3 | What did the attacker do post-access? | Sysmon Event ID 1 | CommandLine (whoami, ipconfig) |
| 4 | Was any defense evasion performed? | Windows Defender Event ID 5001 | Provider, timestamp |
| 5 | What is the full timeline? | All of the above | @timestamp ordering |
| 6 | What is the disposition and response? | Investigation summary | Confirmed C2 — contain and remediate |

---

## Step 1 — Open the alert in Kibana Security

1. Go to **Security → Alerts**
2. Find the alert "Mythic C2 Apollo Agent Detected"
3. Click the alert to expand it

Note the following fields:

| Field | Value |
|---|---|
| `kibana.alert.rule.name` | Mythic C2 Apollo Agent Detected |
| `kibana.alert.rule.severity` | Critical |
| `agent.name` | WIN-KT9TVSF1T0C |
| `winlog.event_data.OriginalFileName` | Apollo.exe |
| `winlog.event_data.Image` | *(full path to apollo.exe)* |
| `winlog.event_data.Hashes` | *(SHA256 of payload)* |
| `@timestamp` | *(when the agent was executed)* |

Screenshot the expanded alert panel — save as
`diagrams/step15-apollo-alert-expanded.png`.

---

## Step 2 — Pivot to Sysmon Event ID 1 (process creation)

Go to **Analytics → Discover** and pull the full process creation event:

```kql
agent.name: "WIN-KT9TVSF1T0C" AND
event.code: "1" AND
winlog.event_data.OriginalFileName: "Apollo.exe"
```

Add these columns to the Discover table:

| Field | What to look for |
|---|---|
| `winlog.event_data.Image` | Full path where the payload was dropped and run |
| `winlog.event_data.CommandLine` | How it was launched |
| `winlog.event_data.ParentImage` | What process spawned it (e.g. PowerShell, explorer.exe) |
| `winlog.event_data.ParentCommandLine` | How the parent was called |
| `winlog.event_data.User` | Which user account ran the payload |
| `winlog.event_data.Hashes` | SHA256 |
| `@timestamp` | Execution time |


Screenshot the Discover result showing all fields — save as
`diagrams/step15-sysmon-event1.png`.

---

## Step 3 — Confirm the C2 beacon (Sysmon Event ID 3)

Query for outbound network connections made by the Apollo process:

```kql
agent.name: "WIN-KT9TVSF1T0C" AND
event.code: "3" AND
winlog.event_data.DestinationIp: "172.31.0.134"
```

Add these columns:

| Field | Expected value |
|---|---|
| `winlog.event_data.Image` | Same Apollo process path from Step 2 |
| `winlog.event_data.DestinationIp` | `172.31.0.134` (Mythic C2 VM4) |
| `winlog.event_data.DestinationPort` | `80` (HTTP C2 profile) |
| `winlog.event_data.Initiated` | `true` (connection was outbound) |
| `@timestamp` | Seconds after process creation — confirms beacon |

**Lab context:** The destination IP `172.31.0.134` is the Mythic C2 server on VM4 in the host-only lab network. In a real environment this would be an external IP requiring threat intel lookup. In this lab it is the confirmed attacker-controlled C2 server set up.


Screenshot — save as `diagrams/step15-c2-beacon-event3.png`.

---

## Step 4 — Identify post-access activity (discovery commands)

Query for process creation events matching the discovery commands run via the Mythic interact window:

```kql
agent.name: "WIN-KT9TVSF1T0C" AND
event.code: "1" AND
winlog.event_data.CommandLine: ("whoami" OR "ipconfig")
```

For each matching event, note:
- The command run
- The timestamp
- The parent process

Screenshot — save as `diagrams/step15-discovery-commands.png`.

---

## Step 5 — Check for defense evasion (Defender disabled)

Query for Windows Defender being disabled — this should have occurred before or shortly after the payload was executed:

```kql
agent.name: "WIN-KT9TVSF1T0C" AND
event.code: "5001"
```

If Event ID 5001 appears before the Apollo execution timestamp, it confirms the attacker deliberately disabled Defender to allow the payload to run without interference.

---

## Step 6 — Build the attack timeline

Using the timestamps collected in Steps 2–5, reconstruct the full kill chain in chronological order:

| Time | Event | Source | MITRE ID |
|---|---|---|---|
| Jun 7, 2026 @ 20:20:17.172 | Windows Defender disabled | Event ID 5001 | T1562.001 |
| Jun 7, 2026 @ 20:26:30.402 | Apollo agent executed | Sysmon Event ID 1 | T1204.002 |
| Jun 7, 2026 @ 20:26:35.567 | C2 beacon to 172.31.0.134 | Sysmon Event ID 3 | T1071.001 |
| Jun 7, 2026 @ 20:28:02.908 | `whoami` run via C2 | Sysmon Event ID 1 | T1033 |
| Jun 7, 2026 @ 20:28:32.124 | `ipconfig` run via C2 | Sysmon Event ID 1 | T1016 |


This timeline is the core of the investigation write-up. It shows the full attack chain in order — exactly what a SOC analyst would submit as evidence in a real incident report.

---

## Step 7 — Disposition

| Property | Value |
|---|---|
| Alert rule | Mythic C2 Apollo Agent Detected |
| Alert severity | Critical |
| Payload | apollo.exe |
| OriginalFileName | Apollo.exe |
| SHA256 | *6FC88C6EE1298FA03ADE7125E2E040C3946C8535ACCB455EB16225C5EE55A3DD* |
| C2 server | 172.31.0.134 (Mythic VM4 — lab-controlled) |
| C2 protocol | HTTP port 80 |
| Post-access activity | whoami, ipconfig |
| Defense evasion | Windows Defender disabled (Event ID 5001) |
| Successful C2 session | ✅ Confirmed — active callback in Mythic UI |
| Disposition | **True Positive — Confirmed C2 Compromise** |
| Response (lab) | No remediation required — controlled simulation |
| Response (real SOC) | Isolate endpoint, kill process, hunt for lateral movement, reimage |

---

## Step 8 — Close the osTicket ticket

Go to `http://localhost/osTicket/scp/` and find the ticket generated by this alert. Add an internal note:

```
INVESTIGATION FINDINGS — Mythic C2 Apollo Agent

Alert: Mythic C2 Apollo Agent Detected
Severity: Critical
Disposition: TRUE POSITIVE — Confirmed C2 compromise

Timeline:
1. Windows Defender disabled (Event ID 5001) — T1562.001
2. Apollo agent executed from Desktop — T1204.002
3. Outbound C2 beacon to 172.31.0.134 confirmed — T1071.001
4. Discovery commands run via C2 session — T1033, T1016

SHA256: 6FC88C6EE1298FA03ADE7125E2E040C3946C8535ACCB455EB16225C5EE55A3DD

Lab context: Controlled simulation. All activity was intentional.
In a real environment: isolate endpoint, kill process, reimage.

Closed by: Rohan S Nair
```

Change ticket status to **Resolved**.

Screenshot the closed ticket — save as `diagrams/step15-osticket-closed.png`.

---

## Investigation write-up

The formal investigation report is documented in:
```
investigations/mythic-c2-investigation.md
```



---

## Key takeaway
A Critical severity C2 detection alert is the most serious alert type in most SOC environments — it means a post-exploitation tool has successfully executed on an endpoint and established an interactive session with an attacker-controlled server. The investigation workflow here — alert → process creation → network beacon → post-access activity → timeline → disposition — is the standard C2 investigation playbook. The fact that this entire chain is visible in Kibana through Sysmon telemetry, with every stage mapped to a MITRE ATT&CK technique, is what good endpoint visibility looks like. In a real SOC, the response to a confirmed True Positive at this severity would be immediate: isolate the endpoint from the network, kill the malicious process, preserve forensic evidence, and initiate a full incident response workflow.

---

## VM status at end of step

| VM | Status |
|---|---|
| ELK + Fleet (VM1) | ✅ Running — Elasticsearch :9200, Kibana :5601, Fleet :8220 |
| Windows target (VM2) | ⏸ Suspended |
| Ubuntu target (VM3) | ⏸ Suspended |
| Mythic C2 (VM4) | ⏸ Suspended |
| osTicket (host) | ✅ Running — C2 investigation ticket resolved |