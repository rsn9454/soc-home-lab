# Attack Scenarios

This document maps every attack simulated in the lab to the MITRE ATT&CK framework. It is built up progressively as each attack phase is completed — from brute force detection through to the full Mythic C2 chain.

Each scenario includes the attack description, the tools used, the detection method, and a link to the corresponding investigation write-up where applicable.

---

## Overview

| # | Scenario | Target |
|---|---|---|
| 1 | SSH Brute Force | Ubuntu Server VM3 | 
| 2 | RDP Brute Force | Windows Server VM2 | 
| 3 | Mythic C2 Full Attack Chain | Windows Server VM2 | 

---

## Scenario 1 — SSH Brute Force

### Description
Repeated automated SSH login attempts against the Ubuntu Server target using incorrect credentials. The goal is to gain initial access to the Linux endpoint via password guessing.

### Attack details
| Property | Value |
|---|---|
| Target | Ubuntu Server VM3 |
| Protocol | SSH (port 22) |
| Tool | Manual attempts |
| Credential type | Username + password guessing |
| Steps performed | Traffic generation + detection rule |

### MITRE ATT&CK mapping
| Tactic | Technique | ID | Notes |
|---|---|---|---|
| Credential Access | Brute Force: Password Guessing | T1110.001 | Repeated failed SSH attempts from single user |
| Initial Access | Valid Accounts | T1078 | Successful SSH login after correct credentials found |

### Detection Rule
| Property | Value |
|---|---|
| Rule name | SSH Brute Force Activity - ubuntutarget |
| Rule type | Threshold |
| Query | `agent.name: "ubuntutarget" AND system.auth.ssh.event: "Failed"` |
| Threshold | Count > 5 from same `user.name` within 2 minutes |
| Severity | Low |
| Rule file | `detection-rules/ssh-brute-force-rule.json` |

### Key log fields
```kql
# Detection query
agent.name: "ubuntutarget" AND system.auth.ssh.event: "Failed"

# Key fields in the alert
system.auth.ssh.event   → "Failed" / "Accepted"
source.ip               → attacker IP
user.name               → username attempted
source.geo.country_name → origin country (external IPs only)
```

### Investigation write-up
`investigations/ssh-brute-force-investigation.md` *(to be completed in later steps)*

---

## Scenario 2 — RDP Brute Force

### Description
Repeated automated RDP login attempts against the Windows Server target using incorrect credentials. Windows event code `4625` with LogonType `10` or `3` is the key telemetry signal.

### Attack details
| Property | Value |
|---|---|
| Target | Windows Server VM2  |
| Protocol | RDP (port 3389) |
| Tool | Manual RDP attempts  |
| Credential type | Username + password guessing |
| Steps performed | Traffic generation + detection rule |

### MITRE ATT&CK mapping
| Tactic | Technique | ID | Notes |
|---|---|---|---|
| Credential Access | Brute Force: Password Guessing | T1110.001 | Repeated event.code 4625 with LogonType 10 |
| Initial Access | Valid Accounts: Local Accounts | T1078.003 | Successful RDP login used for C2 attack chain |

### Detection
| Property | Value |
|---|---|
| Rule name | RDP Brute Force Activity - WIN-KT9TVSF1T0C |
| Rule type | Threshold |
| Query | `agent.name: "WIN-KT9TVSF1T0C" AND event.code: "4625" AND (winlog.event_data.LogonType: "10" OR winlog.event_data.LogonType: "3")` |
| Threshold | Count > 5 from same `source.ip` within 2 minutes |
| Severity | Low |
| Rule file | `detection-rules/rdp-brute-force-rule.json` |

### Key log fields
```kql
# Detection query
agent.name: "WIN-KT9TVSF1T0C" AND
event.code: "4625" AND
(winlog.event_data.LogonType: "10" OR winlog.event_data.LogonType: "3")

# Key fields in the alert
event.code                           → 4625 (failed logon)
winlog.event_data.LogonType          → 10 (RDP/RemoteInteractive) or 3 (in case of local network)
winlog.event_data.TargetUserName     → username attempted
source.ip                            → attacker IP
winlog.event_data.WorkstationName    → attacker machine name

# Successful RDP login (post-brute force)
event.code: "4624" AND (winlog.event_data.LogonType: "10" OR winlog.event_data.LogonType: "3")
```

### Investigation write-up
`investigations/rdp-brute-force-investigation.md` *(to be completed in later steps)*

---

## Scenario 3 — Mythic C2 Full Attack Chain

### Description
A full end-to-end attack simulation using the Mythic C2 framework with an Apollo agent. The attack begins with RDP brute force for initial access, followed by host discovery, defense evasion (disabling Windows Defender), and establishing a persistent C2 beacon back to the Mythic server.

This is the most complex scenario in the lab and maps to multiple MITRE ATT&CK tactics across the full kill chain.

### Attack details
| Property | Value |
|---|---|
| Target | Windows Server VM2 |
| Attacker | Mythic C2 VM4, Kali Linux VM |
| C2 Framework | Mythic |
| Agent | Apollo (Windows x64) |
| C2 profile | HTTP |
| Steps performed | Planning, RDP Brute Force, Mythic install, Execution, Detection, Investigation |

### Attack chain diagram

![SOC Lab Diagram](diagrams/attack-diagram.png)

### MITRE ATT&CK mapping

| # | Stage | Action | Tactic | Technique | ID |
|---|---|---|---|---|---|
| 1 | Initial Access | RDP brute force password guessing | Credential Access | Brute Force: Password Guessing | T1110.001 |
| 2 | Initial Access | Successful RDP login with compromised credentials | Initial Access | Valid Accounts: Local Accounts | T1078.003 |
| 3 | Defense Evasion | Disable Windows Defender real-time protection | Defense Evasion | Impair Defenses: Disable or Modify Tools | T1562.001 |
| 4 | Execution | Apollo agent executed on target | Execution | User Execution: Malicious File | T1204.002 |
| 5 | Command & Control | Apollo HTTP beacon to Mythic C2 | Command & Control | Application Layer Protocol: Web Protocols | T1071.001 |
| 6 | Discovery | `whoami` — identify current user context | Discovery | System Owner/User Discovery | T1033 |
| 7 | Discovery | `ipconfig` — enumerate network config | Discovery | System Network Configuration Discovery | T1016 |
| 8| Discovery | `net user` — enumerate local accounts | Discovery | Account Discovery: Local Account | T1087.001 |
| 9 | Discovery | `net group` — enumerate domain groups | Discovery | Account Discovery: Domain Account | T1087.002 |

### Key log queries (Sysmon telemetry)

```kql
# Apollo agent process creation (Sysmon Event ID 1)
event.code: "1" AND
winlog.channel: "Microsoft-Windows-Sysmon/Operational" AND
winlog.event_data.Image: "*svchost*"

# C2 network beacon (Sysmon Event ID 3)
event.code: "3" AND
winlog.channel: "Microsoft-Windows-Sysmon/Operational" AND
winlog.event_data.DestinationPort: "80" AND
winlog.event_data.Image: "*svchost*"

# Windows Defender disabled (Event ID 1116 / PowerShell)
event.code: "1" AND
winlog.event_data.CommandLine: "*DisableRealtimeMonitoring*"

# Discovery commands run via C2 session
event.code: "1" AND
winlog.event_data.CommandLine: ("*whoami*" OR "*ipconfig*" OR "*net user*" OR "*net group*")
```
Added to `resources/kql-cheatsheet.md`

### Investigation write-up
`investigations/mythic-c2-investigation.md` *(to be completed in later steps)*

---


## References

- [MITRE ATT&CK Framework](https://attack.mitre.org/)
- [Mythic C2 Documentation](https://docs.mythic-c2.website/)
- [Apollo Agent GitHub](https://github.com/MythicAgents/Apollo)
- [Sysmon Event ID Reference](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon)
- [Windows Security Event Log Reference](https://learn.microsoft.com/en-us/windows/security/threat-protection/auditing/basic-audit-logon-events)