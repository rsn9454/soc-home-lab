# Investigation — Mythic C2 Apollo Agent Compromise

## Overview

| Property | Value |
|---|---|
| Investigation ID | INC-001 |
| Alert rule | Mythic C2 Apollo Agent Detected |
| Severity | Critical |
| Date detected | *Jun 7, 2026 @ 20:30:11.381* |
| Analyst | Rohan S Nair |
| Status | ✅ Resolved |
| Disposition | True Positive — Confirmed C2 compromise (controlled lab simulation) |

---

## Alert details

| Property | Value |
|---|---|
| Rule name | Mythic C2 Apollo Agent Detected |
| Rule type | Custom query |
| Detection query | `event.code: 1 AND (winlog.event_data.Hashes: *6FC88C6EE1298FA03ADE7125E2E040C3946C8535ACCB455EB16225C5EE55A3DD* OR winlog.event_data.OriginalFileName: Apollo.exe)` |
| Severity | Critical |
| Risk score | 99 |
| MITRE techniques | T1204.002, T1071.001 |
| Agent name | WIN-KT9TVSF1T0C |
| Alert timestamp | *Jun 7, 2026 @ 20:30:11.381* |

---

## Affected host

| Property | Value |
|---|---|
| Hostname | WIN-KT9TVSF1T0C |
| Operating system | Windows Server 2022 |
| IP address | 172.31.0.20 |
| Role | Target endpoint (VM2) |
| Elastic Agent status | Enrolled |
| Sysmon installed | Yes — Olaf config |

---

## Question 1 — What process triggered the alert?

**Data source:** Sysmon Event ID 1 (process creation)

**Query used:**
```kql
event.code: 1 AND 
(winlog.event_data.Hashes: *6FC88C6EE1298FA03ADE7125E2E040C3946C8535ACCB455EB16225C5EE55A3DD*
OR winlog.event_data.OriginalFileName: Apollo.exe)
```

**Findings:**

| Field | Value |
|---|---|
| `winlog.event_data.Image` | *C:\Users\Public\Downloads\apollo.exe* |
| `winlog.event_data.OriginalFileName` | Apollo.exe |
| `winlog.event_data.CommandLine` | *"C:\Users\Public\Downloads\apollo.exe"* |
| `winlog.event_data.ParentImage` | *C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe* |
| `winlog.event_data.ParentCommandLine` | *"C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe"* |
| `winlog.event_data.User` | *WIN-KT9TVSF1T0C\Administrator* |
| `winlog.event_data.Hashes` | SHA256=*6FC88C6EE1298FA03ADE7125E2E040C3946C8535ACCB455EB16225C5EE55A3DD* |
| `@timestamp` | *Jun 7, 2026 @ 20:26:30.402* |

**Analysis:**

The payload `apollo.exe` was executed from the Administrator's Desktop. The `OriginalFileName` field confirms the binary is the Apollo C2 agent — this field is embedded in the PE header at compile time and cannot be changed by renaming the file. The SHA256 hash uniquely identifies this specific Mythic-generated payload build.

The parent process indicates the payload was launched via *PowerShell*.

**MITRE mapping:** T1204.002 — User Execution: Malicious File

---

## Question 2 — Did it establish a network connection?

**Data source:** Sysmon Event ID 3 (network connection)

**Query used:**
```kql
agent.name: "WIN-KT9TVSF1T0C" AND
event.code: "3" AND
winlog.event_data.DestinationIp: "172.31.0.134"
```

**Findings:**

| Field | Value |
|---|---|
| `winlog.event_data.Image` | *C:\Users\Public\Downloads\apollo.exe* |
| `winlog.event_data.DestinationIp` | 172.31.0.134 |
| `winlog.event_data.DestinationPort` | 80 |
| `winlog.event_data.Initiated` | true (outbound connection) |
| `winlog.event_data.Protocol` | tcp |
| `@timestamp` | *20:26:35.567* |


**Analysis:**

Within seconds of execution, the Apollo agent established an outbound HTTP connection to `172.31.0.134:80` — the Mythic C2 server on VM4. The `Initiated: true` field confirms this connection was initiated by the process on the endpoint, not an inbound connection.

Multiple Event ID 3 entries appear at regular 10-second intervals — this is the C2 beacon pattern. In a real environment, consistent outbound connections from a process at a fixed interval to an external IP are a strong indicator of C2 beacon behaviour and warrant immediate investigation.

**Lab context:** `172.31.0.134` is the Mythic C2 server on the lab host-only network (VM4). In a real SOC this would be an external IP requiring threat intelligence lookup.

**MITRE mapping:** T1071.001 — Application Layer Protocol: Web Protocols

---

## Question 3 — What did the attacker do post-access?

**Data source:** Sysmon Event ID 1 — discovery commands run via C2 interact window

**Query used:**
```kql
agent.name: "WIN-KT9TVSF1T0C" AND
event.code: "1" AND
winlog.event_data.CommandLine: (*whoami* OR *ipconfig* OR *net user*)
```

**Findings:**

| Command | Timestamp | Purpose | MITRE |
|---|---|---|---|
| `whoami` | *Jun 7, 2026 @ 20:28:01.607* | Identify current user context | T1033 |
| `ipconfig` | *Jun 7, 2026 @ 20:28:31.830* | Enumerate network configuration | T1016 |
| `net user` | *Jun 7, 2026 @ 20:28:49.680* | Enumerate local user accounts | T1087.001 |

**Analysis:**
The attacker ran standard host discovery commands immediately after establishing the C2 session — a textbook initial reconnaissance pattern post-compromise. The parent process for each command was the Apollo agent process, confirming these commands were issued via the Mythic interact window rather than by a legitimate user.

This discovery activity would allow the attacker to determine:
- Whether they have Administrator privileges (`whoami`)
- What network the host is on and what other hosts may be reachable (`ipconfig`)
- What user accounts exist and could be targeted for privilege escalation (`net user`)

**MITRE mappings:** T1033, T1016, T1087.001

---

## Question 4 — Was any defense evasion performed?

**Data source:** Windows Defender Event ID 5001

**Query used:**
```kql
agent.name: "WIN-KT9TVSF1T0C" AND
event.code: "5001"
```

**Findings:**

| Field | Value |
|---|---|
| Event ID | 5001 |
| Provider | Microsoft-Windows-Windows Defender |
| Meaning | Real-time protection disabled |
| Timestamp | *Jun 7, 2026 @ 20:20:17.172* |

**Analysis:**
Windows Defender real-time protection was disabled before the Apollo agent was executed. This is a deliberate defense evasion step — without it, Defender would detect and quarantine the Apollo payload before it could run. The sequence of Defender disabled → payload executed is a classic pre-execution evasion pattern.

**MITRE mapping:** T1562.001 — Impair Defenses: Disable or Modify Tools

---

## Attack timeline

Events in chronological order based on `@timestamp` from Kibana Discover:

| # | Time | Event | Log source | Event ID | MITRE |
|---|---|---|---|---|---|
| 1 | Jun 7, 2026 @ 20:20:17.172 | Windows Defender real-time protection disabled | Windows Defender | 5001 | T1562.001 |
| 2 | Jun 7, 2026 @ 20:26:30.402 | apollo.exe executed on Desktop | Sysmon | 1 | T1204.002 |
| 3 | Jun 7, 2026 @ 20:26:35.567 | Outbound C2 beacon to 172.31.0.134:80 | Sysmon | 3 | T1071.001 |
| 4 | Jun 7, 2026 @ 20:28:01.607 | `whoami` run via C2 session | Sysmon | 1 | T1033 |
| 5 | Jun 7, 2026 @ 20:28:31.830 | `ipconfig` run via C2 session | Sysmon | 1 | T1016 |
| 6 | Jun 7, 2026 @ 20:28:49.680 | `net user` run via C2 session | Sysmon | 1 | T1087.001 |

---

## MITRE ATT&CK summary

| Tactic | Technique | ID | Observed |
|---|---|---|---|
| Credential Access | Brute Force: Password Guessing | T1110.001 | RDP brute force for initial access |
| Initial Access | Valid Accounts: Local Accounts | T1078.003 | Successful RDP login post-brute force |
| Defense Evasion | Impair Defenses: Disable or Modify Tools | T1562.001 | Defender disabled before payload execution |
| Execution | User Execution: Malicious File | T1204.002 | Apollo agent executed from Desktop |
| Command & Control | Application Layer Protocol: Web Protocols | T1071.001 | HTTP beacon to 172.31.0.134:80 |
| Discovery | System Owner/User Discovery | T1033 | `whoami` via C2 session |
| Discovery | System Network Configuration Discovery | T1016 | `ipconfig` via C2 session |
| Discovery | Account Discovery: Local Account | T1087.001 | `net user` via C2 session |

---

## Evidence

| Evidence item | Location | Description |
|---|---|---|
| Apollo process creation | Sysmon Event ID 1 | OriginalFileName: Apollo.exe, SHA256 hash |
| C2 network beacon | Sysmon Event ID 3 | Outbound to 172.31.0.134:80, Initiated: true |
| Discovery commands | Sysmon Event ID 1 | whoami, ipconfig, net user via Apollo parent |
| Defender disabled | Windows Defender Event ID 5001 | Real-time protection off before execution |
| Kibana alert | Security → Alerts | Critical severity rule fired on single match |
| Attack diagram | `diagrams/attack-diagram.png` | Full kill chain planned beforehand |

---

## Disposition

| Property | Value |
|---|---|
| Verdict | **True Positive** |
| Confidence | High — OriginalFileName + SHA256 hash match known C2 agent |
| Scope | Single endpoint (WIN-KT9TVSF1T0C) |
| Data exfiltration | Not observed in lab simulation |
| Persistence mechanisms | Not observed in lab simulation |
| Lateral movement | Not observed in lab simulation |

---

## Response actions

**In this lab (controlled simulation):**
No remediation required — all activity was intentional and executed by me as part of the project.

**In a real SOC environment, the following actions would be taken immediately:**

1. **Contain** — isolate the endpoint from the network to prevent lateral movement and cut the C2 beacon
2. **Kill** — terminate the Apollo agent process (`apollo.exe`)
3. **Preserve** — capture a memory image and disk snapshot for forensic analysis before any remediation
4. **Hunt** — search all other enrolled endpoints for the same SHA256 hash and `OriginalFileName: Apollo.exe` to check for lateral spread
5. **Block** — add the payload SHA256 to endpoint blocklists; block outbound connections to the C2 IP at the network perimeter
6. **Escalate** — notify the incident response team and open a formal P1 incident ticket
7. **Remediate** — reimage the compromised endpoint
8. **Harden** — review how Defender was disabled and enforce tamper protection via Group Policy

---

## Lessons learned

- **Sysmon is essential.** Without the Olaf config installed, neither the process creation (Event ID 1) nor the C2 beacon (Event ID 3) would have been visible in the SIEM. Default Windows logging would have produced almost no useful telemetry for this attack.

- **`OriginalFileName` beats `Image` for detection.** Querying by file path (`Image`) would fail if the attacker renamed the payload. The PE header field `OriginalFileName` cannot be changed without recompiling — it is the more durable detection signal.

- **Beacon interval is a detection signal.** The regular 10-second outbound connections to a single destination IP are a classic C2 pattern. Anomaly detection or periodic connection analysis would catch this even without a hash-based rule.

- **Defense evasion precedes execution.** Defender was disabled before the payload ran — not after. A detection for Defender being disabled should be treated as a leading indicator of imminent malicious execution, not just a standalone policy violation.

---

## Investigator sign-off

| Field | Value |
|---|---|
| Investigator | Rohan S Nair |
| Date closed | *June 7* |
| osTicket ID | *#851442* |
| Related files | `detection-rules/mythic-c2-apollo-process-creation.json` |
| | `dashboards/mythic-c2-activity-dashboard.ndjson` |
| | `ATTACK-SCENARIOS.md — Scenario 3` |