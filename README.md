# 🛡️ SOC Home Lab

## Overview

A self-directed SOC analyst home lab built across 16 documented steps — covering full SIEM deployment, endpoint telemetry, detection engineering, C2 attack simulation, and incident investigation using open-source tooling on a constrained 8 GB local machine.

Every step is documented with the intent of understanding not just how to configure tools, but why each configuration decision matters in a real SOC context.

---

## Architecture

![SOC Lab Logical Diagram](diagrams/soc-lab-logical-diagram.png)

---

## Lab environment

| Component | OS | RAM | Purpose |
|---|---|---|---|
| ELK + Fleet Server (VM1) | Ubuntu 22.04 | 3 GB | SIEM core — Elasticsearch, Kibana, Fleet |
| Windows Server 2022 (VM2) | Windows Server 2022 | 2 GB | RDP target, Sysmon endpoint |
| Ubuntu Server (VM3) | Ubuntu 22.04 | 2 GB | SSH target |
| Mythic C2 Server (VM4) | Ubuntu 22.04 | 2 GB | Attack simulation (steps 11–15) |
| osTicket | Windows 11 host (XAMPP) | ~200 MB | SOC ticketing system |
| Hypervisor | VMware | — | Host: Windows 11, 8 GB RAM |

**RAM management:** VMs are booted selectively — never all simultaneously — to stay within the 8 GB constraint. osTicket runs natively on the Windows host to eliminate a full VM.

---

## Tools and technologies

| Category | Tools |
|---|---|
| SIEM | Elasticsearch 8.x, Kibana, Fleet Server |
| Endpoint agents | Elastic Agent, Sysmon (Olaf config) |
| EDR | Elastic Defend |
| C2 framework | Mythic, Apollo agent (HTTP profile) |
| Ticketing | osTicket (XAMPP) |
| Hypervisor | VMware |
| Query language | KQL, EQL |
| Framework | MITRE ATT&CK |

---

## Project steps

| Step | Topic | Report | Screenshots |
|------|-------|--------|-------------|
| 01 | Logical diagram — lab architecture design | [Read](steps/step01-logical-diagram.md) | [View](diagrams/soc-lab-logical-diagram.png) |
| 02 | Install Elasticsearch on VM1 | [Read](steps/step02-elasticsearch.md) | — |
| 03 | Install Kibana on VM1 | [Read](steps/step03-kibana.md) | [View](diagrams/step03-kibana-ui.png) |
| 04 | Set up Fleet Server on VM1 | [Read](steps/step04-fleet-server.md) | [View](diagrams/step04-fleet-enrolled.png) |
| 05 | Enroll Windows Server 2022 target via Fleet | [Read](steps/step05-windows-agent.md) | [View](diagrams/step05-windows-enrolled.png) |
| 06 | Install and configure Sysmon on VM2 | [Read](steps/step06-sysmon.md) | [View](diagrams/step06-sysmon-logs.png) |
| 07 | Enroll Ubuntu Server target via Fleet | [Read](steps/step07-ubuntu-agent.md) | [View](diagrams/step07-ubuntu-enrolled.png) |
| 08 | Create SSH brute force detection rule | [Read](steps/step08-ssh-alert-rule.md) | [View](diagrams/step08-ssh-alert-triggered.png) |
| 09 | Create RDP brute force detection rule | [Read](steps/step09-rdp-alert-rule.md) | [View](diagrams/step09-rdp-alert-triggered.png) |
| 10 | Design C2 attack chain diagram + MITRE mapping | [Read](steps/step10-attack-diagram.md) | [View](diagrams/attack-diagram.png) |
| 11 | Install Mythic C2 server on VM4 | [Read](steps/step11-mythic-server-install.md) | [View](diagrams/step11-mythic-ui.png) |
| 12 | Generate Apollo payload, execute attack chain, confirm C2 callback | [Read](steps/step12-apollo-payload-and-c2-callback.md) | [View](diagrams/) |
| 13 | Build Apollo detection rule + 3-panel C2 activity dashboard | [Read](steps/step13-c2-detection-and-dashboard.md) | [View](diagrams/) |
| 14 | Install osTicket + Kibana webhook integration | [Read](steps/step14-osticket-install.md) | [View](diagrams/step14-test-ticket-created.png) |
| 15 | Investigate Mythic C2 Apollo agent alert end-to-end | [Read](steps/step15-mythic-agent-investigation.md) | [View](investigations/mythic-c2-investigation.md) |
| 16 | Install and test Elastic Defend (EDR) on Windows target | [Read](steps/step16-elastic-defend.md) | [View](diagrams/) |

---

## Attack scenarios and MITRE ATT&CK coverage

Full mapping document: [`ATTACK-SCENARIOS.md`](ATTACK-SCENARIOS.md)

| Scenario | Target | Technique | ID | Detection |
|---|---|---|---|---|
| SSH brute force | Ubuntu VM3 | Brute Force: Password Guessing | T1110.001 | `detection-rules/ssh-brute-force-rule.json` |
| RDP brute force | Windows VM2 | Brute Force: Password Guessing | T1110.001 | `detection-rules/rdp-brute-force-rule.json` |
| RDP initial access | Windows VM2 | Valid Accounts: Local Accounts | T1078.003 | RDP rule (event.code 4624) |
| Defender disabled | Windows VM2 | Impair Defenses: Disable Tools | T1562.001 | C2 dashboard — Event ID 5001 |
| Apollo agent executed | Windows VM2 | User Execution: Malicious File | T1204.002 | `detection-rules/mythic-c2-apollo-process-creation.json` |
| C2 HTTP beacon | Windows VM2 | App Layer Protocol: Web | T1071.001 | C2 dashboard — Sysmon Event ID 3 |
| System discovery | Windows VM2 | System Owner/User Discovery | T1033 | Sysmon Event ID 1 + CommandLine |
| Network discovery | Windows VM2 | Network Configuration Discovery | T1016 | Sysmon Event ID 1 + CommandLine |
| Account discovery | Windows VM2 | Account Discovery: Local | T1087.001 | Sysmon Event ID 1 + CommandLine |

---

## Detection rules

| Rule | Type | Severity | File |
|---|---|---|---|
| SSH Brute Force Activity - ubuntutarget | Threshold | Low | [`detection-rules/ssh-brute-force-rule.json`](detection-rules/ssh-brute-force-rule.json) |
| RDP Brute Force Activity - WIN-KT9TVSF1T0C | Threshold | Low | [`detection-rules/rdp-brute-force-rule.json`](detection-rules/rdp-brute-force-rule.json) |
| Mythic C2 Apollo Agent Detected | Custom query | Critical | [`detection-rules/mythic-c2-apollo-process-creation.json`](detection-rules/mythic-c2-apollo-process-creation.json) |

---

## Investigation write-ups

| Incident | File | Disposition |
|---|---|---|
| Mythic C2 Apollo agent | [`investigations/mythic-c2-investigation.md`](investigations/mythic-c2-investigation.md) | True Positive — full C2 kill chain confirmed and documented |

---

## Key artifacts

```
detection-rules/            ← Exported Kibana rule JSONs (importable)
dashboards/                 ← Exported Kibana dashboard ndjson files
investigations/             ← Formal SOC investigation write-ups
configs/sysmon/             ← Olaf Sysmon config XML
diagrams/                   ← Screenshots of steps
resources/kql-cheatsheet.md ← KQL/EQL queries built during the project
ATTACK-SCENARIOS.md         ← Full MITRE ATT&CK mapping for all scenarios
```

---

## Skills demonstrated

- Deploying and tuning a single-node ELK Stack (Elasticsearch 8.x + Kibana + Fleet Server) under a 3 GB RAM constraint — JVM heap capping, swapfile configuration, UFW rules
- Enrolling Windows Server 2022 and Ubuntu Server endpoints via Elastic Agent Fleet, with Sysmon and Windows Event Log integrations
- Writing KQL threshold detection rules for SSH and RDP brute force (T1110.001) with MITRE ATT&CK tagging
- Executing a full Mythic C2 attack chain: RDP brute force → initial access → discovery commands → defense evasion → C2 beacon — and detecting every stage via Sysmon telemetry
- Building a custom Kibana detection rule using `winlog.event_data.OriginalFileName` and SHA256 hash — surviving payload file-rename attacks
- Constructing a 3-panel C2 activity dashboard: process creation (Event ID 1), outbound beacons (Event ID 3), Defender disabled (Event ID 5001)
- Integrating Kibana alert webhooks with osTicket to auto-generate investigation tickets on every rule fire
- Conducting structured alert investigations using a 4-question framework: source context → scope → success status → disposition
- Configuring Elastic Defend EDR in prevention mode and verifying it blocks the Apollo payload that previously executed freely

---

## Motivation

Built this lab to develop practical blue team skills beyond certification study — specifically to gain hands-on experience with SIEM deployment, detection rule engineering, and full attack simulation in a controlled environment. The goal was to understand not just how to use these tools, but how they fit together in a real SOC workflow: from raw endpoint log to fired alert to investigation ticket to closed incident.

---

## References

- [Elasticsearch documentation](https://www.elastic.co/docs)
- [Sysmon — Olaf config](https://github.com/olafhartong/sysmon-modular)
- [Mythic C2 framework](https://github.com/its-a-feature/Mythic)
- [Apollo agent](https://github.com/MythicAgents/Apollo)
- [MITRE ATT&CK framework](https://attack.mitre.org/)
- [osTicket](https://osticket.com/)