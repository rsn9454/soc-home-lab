# Step 01 — Logical diagram

## Objective
Design and document the full architecture of the SOC home lab before touching a single VM — establishing what every component is, why it exists, and how they communicate with each other on a constrained 8GB RAM machine.

---

## Lab architecture overview

The lab consists of six logical components spread across one physical Windows host and four VMware VMs, all connected via a host-only network.

| Component | Type | OS | RAM | Purpose |
|---|---|---|---|---|
| Analyst workstation | Physical host | Windows (host) | — | Browser access to Kibana, osTicket |
| ELK + Fleet Server | VM1 | Ubuntu 22.04 | 3 GB | SIEM core — Elasticsearch, Kibana, Fleet |
| Windows Server target | VM2 | Windows Server 2022 | 2 GB | RDP target, Sysmon endpoint |
| Ubuntu Server target | VM3 | Ubuntu 22.04 | 1 GB | SSH target, Elastic Agent endpoint |
| Mythic C2 server | VM4 | Ubuntu 22.04 | 1 GB | Attack simulation (steps 21–26 only) |
| osTicket | Physical host | Windows (XAMPP) | ~200 MB | Ticketing — receives Kibana alert webhooks |

---

## Network design

All VMs are connected to a **VMware host-only network** (`172.31.0.0/24`).

- The Windows host can reach all VMs via the host-only adapter
- VMs can communicate with each other on the same subnet
- No VM is directly exposed to the internet (intentional for lab safety)
- The ELK VM also has a NAT adapter for package installation

### Port reference

| Service | VM | Port |
|---|---|---|
| Elasticsearch | VM1 | 9200 |
| Kibana | VM1 | 5601 |
| Fleet Server | VM1 | 8220 |
| RDP | VM2 | 3389 |
| SSH | VM3 | 22 |
| Mythic C2 UI | VM4 | 7443 |
| osTicket | Host (XAMPP) | 80 |

---

## Data flow

```
[Windows target / Ubuntu target]
        |
        | Elastic Agent (logs, events, Sysmon telemetry)
        v
[Fleet Server :8220]  ──>  [Elasticsearch :9200]
                                    |
                              [Kibana :5601]
                                    |
                    ┌───────────────┴──────────────┐
                    |                              |
             [Analyst browser]          [Alert webhook]
             (step investigation)              |
                                        [osTicket :80]
                                        (auto-ticket created)


[Mythic C2 VM4]  - - - - C2 beacon - - - ->  [Windows target VM2]

```

---

## Tools used

- **draw.io** — diagram created and exported as `.png`
- **VMware** — hypervisor for all four VMs
- **XAMPP** — Apache + MySQL + PHP stack on Windows host for osTicket

---

## Diagram

![SOC Lab Logical Diagram](../diagrams/soc-lab-logical-diagram.png)

> Diagram shows: Windows host boundary, VMware host-only network boundary, all six components with RAM and OS labels, data flow arrows (Elastic Agent logs, Kibana webhook to osTicket, Mythic C2 beacon), and color coding by component type.

---

## Challenges / notes

- osTicket was moved to the Windows host (via XAMPP) instead of a dedicated VM — this removes one VM and saves 1 GB of RAM, which is significant on an 8 GB machine
- Elasticsearch JVM heap will be capped at 1 GB (`-Xms1g -Xmx1g`) to keep the ELK VM within its 3 GB budget
- A 2 GB swapfile will be added to the ELK VM to prevent OOM kills during heavy log ingestion

---

## Key takeaway

Designing the architecture before building prevents wasted effort — knowing the RAM constraints upfront shaped every decision, from co-locating Fleet Server with ELK, to running osTicket natively on the host. In a real SOC environment, this phase is called **solution design** and is done before any infrastructure is provisioned.

