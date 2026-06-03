# Step 10 — Create Attack Diagram

## Objective
Design and document the full Mythic C2 attack chain as a visual diagram before any hands-on execution begins — mapping every stage of the planned attack from initial access through to C2 beacon establishment.

---

## Prerequisites
- SSH and RDP brute force detection rules are active
- No VMs need to be running for this step
- draw.io installed or accessible at [draw.io](https://app.diagrams.net)

---

## Why diagram before attacking

Documenting the attack plan before execution is standard red team practice. It:

- Forces you to think through each stage before touching a keyboard
- Gives you a reference to check off stages as you execute them
- Mirrors what real threat actors do — attacks are planned, not improvised

---

## Attack chain overview

The diagram covers the following six stages:

| Stage | Action | VM involved |
|---|---|---|
| 1 | Generate Apollo agent payload on Mythic C2 server | VM4 |
| 2 | RDP brute force against Windows target | VM4 → VM2 |
| 3 | Initial access — successful RDP login | VM2 |
| 4 | Discovery — run enumeration commands | VM2 |
| 5 | Defense evasion — disable Windows Defender | VM2 |
| 6 | C2 beacon — Mythic agent calls back to Mythic | VM2 → VM4 |

---

## Diagram

![Attack Chain Diagram](../diagrams/attack-diagram.png)

---

## Key takeaway
Planning the attack chain visually before execution is the habit that separates deliberate security practice from ad-hoc tool running. The diagram produced here will be referenced in every upcoming step — each execution step checks off one stage of this plan. In a real red team engagement, this diagram would be part of the rules of engagement documentation submitted before any testing begins. In the context of this lab, it is the clearest way to show that I understand the full kill chain — not just individual tools.

---


## VM status at end of step

| VM | Status |
|---|---|
| ELK + Fleet (VM1) | ✅ Running — Elasticsearch :9200, Kibana :5601, Fleet :8220 |
| Windows target (VM2) | ⏸ Suspended — needed when attack starts |
| Ubuntu target (VM3) | ⏸ Suspended — not needed until further notice |
| Mythic C2 (VM4) | Not yet created |
| osTicket (host) | Not yet installed |