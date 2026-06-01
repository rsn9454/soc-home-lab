# Step 08 — Create SSH Brute Force Alert Rule

## Objective
Build a threshold-based SSH brute force detection rule in Kibana Security, tag it with the correct MITRE ATT&CK technique, test fire it against real data, and export the rule as JSON for the repo — producing the first working detection artifact of the lab.

---

## Prerequisites
- SSH brute force traffic has been generated against VM3 and logs are visible in Kibana Discover
- `system.auth.ssh.event: "Failed"` events exist in Elasticsearch

---

## Background: what this rule detects

A brute force attack against SSH generates a high volume of failed authentication events from the same user in a short time window. The rule logic is simple:

```
IF count of system.auth.ssh.event: "Failed"
   FROM the same user.name
   EXCEEDS 5
   IN the last 2 minutes
THEN fire an alert
```

This maps directly to **MITRE ATT&CK T1110.001 — Brute Force: Password Guessing**.

---

## Step 1 — Navigate to the rule creation page

In Kibana, go to:
```
Security → Rules → Detection rules (SIEM) → Create new rule
```

---

## Step 2 — Select rule type

Select **Threshold** as the rule type.

> Threshold rules are the correct choice here because brute force is defined by **volume* — a single failed login is normal, fifty in two minutes is an attack. A threshold rule counts matching events and only fires when the count crosses the limit you set.

Click **Continue**.

---

## Step 3 — Define the rule query

In the query field, enter:
```kql
system.auth.ssh.event: "Failed" AND agent.name: "ubuntutarget"
```

Set the index pattern to your logs data view (e.g. `logs-*`).

### Configure the threshold

| Setting | Value |
|---|---|
| Group by field | `user.name` |
| Threshold | 5 |

> Grouping by `user.name` means the rule counts failed attempts **per user separately**. 

Click **Continue**.

---

## Step 4 — Configure rule details

Fill in the rule metadata:

| Field | Value |
|---|---|
| Name | `SSH Brute Force Activity - ubuntutarget` |
| Description | `Alert is generated if a same user fails to login 5+ times to the agent 'ubuntutarget' in the last 2 minutes` |
| Severity | Low|
| Risk score | 21 |
| Tags | `brute-force`, `ssh`, `linux`, `T1110.001` |

### MITRE ATT&CK mapping
Click **Add MITRE ATT&CK threat**:

| Field | Value |
|---|---|
| Tactic | Credential Access |
| Technique | T1110 — Brute Force |
| Sub-technique | T1110.001 — Password Guessing |

Click **Continue**.

---

## Step 5 — Set the schedule

| Setting | Value |
|---|---|
| Runs every | 2 minute |
| Additional look-back time | 0 minutes |

Click **Continue**.

---

## Step 6 — Rule actions (skip for now)

Leave the Actions section empty for now. A connector to osTicket is added in later steps. The rule will still appear in **Security → Alerts** when it fires.

Click **Create & enable rule**.

---

## Step 7 — Verify the rule is active

Go to **Security → Rules → Detection rules (SIEM)**.

Find `SSH Brute Force Activity - ubuntutarget` and confirm:
- **Status:** Active
- **Last run:** shows a recent timestamp (within the last minute)
- **Last response:** Succeeded

---

## Step 8 — Test fire the rule

Trigger enough failed SSH attempts from your Windows host to exceed the threshold:

Wait 1–2 minutes, then go to **Security → Alerts**.

You should see a new alert with:
- **Rule name:** SSH Brute Force Activity - ubuntutarget
- **Severity:** Low
- **Source IP:** 172.31.0.1 (your Windows host's host-only adapter IP)

Click into the alert and explore the fields — note the event details, source IP, and the timeline of failed attempts.

Take a screenshot of the fired alert and save as:
```
diagrams/step08-ssh-alert-triggered.png
```

---

## Export the rule for the repo

1. Go to **Security → Rules → Detection rules (SIEM)**
2. Tick the checkbox next to `SSH Brute Force Activity - ubuntutarget`
3. Click **Bulk actions → Export**
4. Save the downloaded `.ndjson` file as:
   ```
   detection-rules/ssh-brute-force-alert.ndjson
   ```

---

## KQL queries used (add to cheatsheet)

```kql
# Verify alert fired — check Security Alerts index
kibana.alert.rule.name: "SSH Brute Force Activity - ubuntutarget"
```

---

## Rule summary

| Property | Value |
|---|---|
| Rule name | SSH Brute Force Activity - ubuntutarget |
| Rule type | Threshold |
| Query | `system.auth.ssh.event: "Failed" AND agent.name: "ubuntutarget"` |
| Group by | `user.name` |
| Threshold | Count > 5 |
| Check interval | Every 2 minute |
| Severity | Low |
| Risk score | 21 |
| MITRE Tactic | Credential Access |
| MITRE Technique | T1110.001 — Brute Force: Password Guessing |
| Action connector | None (osTicket added in later step) |
| Exported to repo | `detection-rules/ssh-brute-force-alert.json` |

---

## Key takeaway

A threshold rule is the simplest and most effective detection for brute force attacks — it requires no ML model, no baseline, and no complex logic. The power is in the grouping: by counting per `user.name`, the rule correctly identifies a single user hammering the server rather than treating all failed logins globally. This distinction matters in production where many legitimate users occasionally mistype their passwords. The MITRE ATT&CK mapping added here is not decorative — it feeds into Kibana's built-in ATT&CK matrix view, which gives a visual overview of which techniques your detection coverage addresses, and which ones have gaps.

---

## VM status at end of step

| VM | Status |
|---|---|
| ELK + Fleet (VM1) | ✅ Running — Elasticsearch :9200, Kibana :5601, Fleet :8220 |
| Windows target (VM2) | ⏸ Suspended — not needed for this step |
| Ubuntu target (VM3) | ✅ Running — needed to receive SSH test attempts |
| Mythic C2 (VM4) | Not yet created |
| osTicket (host) | Not yet installed |

