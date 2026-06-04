# 🛡️ SOC Home Lab

## Overview
Hands-on SOC lab built step by step using Elasticsearch, Logstash, Kibana, Sysmon, Mythic C2, osTicket

## Architecture
![SOC Lab Diagram](diagrams/soc-lab-logical-diagram.png)

## Local lab setup (8GB RAM)
| VM | OS | RAM | Purpose |
|----|-----|-----|---------|
| ELK + Fleet (VM1) | Ubuntu 22.04 | 3GB | SIEM + agent mgmt |
| Windows target (VM2) | Windows Server 2022 | 2GB | RDP target + Sysmon |
| Ubuntu target (VM3) | Ubuntu 22.04 | 1GB | SSH target |
| Mythic C2 (VM4) | Ubuntu 22.04 | 1GB | Attack sim |
| osTicket | Windows 11 host (XAMPP) | — | Ticketing |

## Tools
Elasticsearch · Kibana · Fleet Server · Elastic Agent · Sysmon · Mythic C2 · osTicket · VirtualBox

## Progress
| Step | Topic | Status | Report | Screenshots |
|------|-------|--------|--------|-------------|
| 1 | Logical diagram | ✅ | [Read](steps/step01-logical-diagram.md) | [View](diagrams/soc-lab-logical-diagram.png) |
| 2 | Install Elasticsearch (VM1) | ✅ | [Read](steps/step02-elasticsearch.md) | - |
| 3 | Install Kibana (VM1) | ✅ | [Read](steps/step03-kibana.md) | [View](diagrams/step03-kibana-ui.png) |
| 4 | Set up Fleet Server (VM1) | ✅ | [Read](steps/step04-fleet-server.md) | [View](diagrams/step04-fleet-enrolled.png) |
| 5 | Install Elastic Agent on Windows Server 2022 (VM2) | ✅ | [Read](steps/step05-windows-agent.md) | [View](diagrams/step05-windows-enrolled.png) |
| 6 | Install and configure Sysmon (VM2) | ✅ | [Read](steps/step06-sysmon.md) | [View](diagrams/step06-sysmon-logs.png) |
| 7 | Install Elastic Agent on Ubuntu Server (VM3) | ✅ | [Read](steps/step07-ubuntu-agent.md) | [View](diagrams/step07-ubuntu-enrolled.png) |
| 8 | Create SSH brute force alert rule | ✅ | [Read](steps/step08-ssh-alert-rule.md) | [View](diagrams/step08-ssh-alert-triggered.png) |
| 9 | Create RDP brute force alert rule | ✅ | [Read](steps/step09-rdp-alert-rule.md) | [View](diagrams/step09-rdp-alert-triggered.png) |
| 10 | Create C2 attack chain diagram | ✅ | [Read](steps/step10-attack-diagram.md) | [View](diagrams/attack-diagram.png) |
| 11 | Install Mythic C2 server (VM4) | ✅ | [Read](steps/step11-mythic-server-install.md) | [View](diagrams/step11-mythic-ui.png) |
| 12 |  Apollo agent + HTTP C2 profile installed, payload executed, callback confirmed (C2 attack chain is completed) | ✅ | [Read](steps/step12-apollo-payload-and-c2-callback.md) | [View](diagrams/) |
| 13 | Apollo C2 detection rule + 3-panel activity dashboard built in Kibana | ✅ | [Read](steps/step13-c2-detection-and-dashboard.md) | [View](diagrams/) |


## Skills demonstrated
- SIEM deployment and configuration
- Detection rule engineering (KQL / EQL)
- Brute force attack detection (SSH + RDP)
- C2 framework detection (Mythic / Apollo agent)
- MITRE ATT&CK mapping
- Incident investigation workflow
- Ticketing system integration
- EDR configuration (Elastic Defend)