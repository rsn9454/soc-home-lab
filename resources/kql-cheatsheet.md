# Windows KQL Queries

```kql
# Event 4624 = successful logon — should appear every time you log into the VM
agent.name: "WIN-KT9TVSF1T0C" AND event.code: "4624"
```
```kql
# All Sysmon events
agent.name: "WIN-KT9TVSF1T0C" AND
event.provider: "Microsoft-Windows-Sysmon"
```
```kql
# All Sysmon process creation events from Windows target
winlog.event_id: "1" AND event.provider: "Microsoft-Windows-Sysmon"

# Sysmon network connections from Windows target
winlog.event_id: "3" AND event.provider: "Microsoft-Windows-Sysmon"

# Sysmon events by a specific process name
winlog.event_id: "1" AND winlog.event_data.Image: "*powershell.exe"

# Sysmon DNS queries (Event ID 22)
winlog.event_id: "22" AND event.provider: "Microsoft-Windows-Sysmon"

# Sysmon file creation events
winlog.event_id: "11" AND event.provider: "Microsoft-Windows-Sysmon"
```
# Linux KQL Queries

```kql
# All logs from Ubuntu target
agent.name: "ubuntutarget"

# SSH authentication events
agent.name: "ubuntutarget" AND event.dataset: "system.auth"

# Failed SSH login attempts
agent.name: "ubuntutarget" AND
event.dataset: "system.auth" AND
(system.auth.ssh.event: "Invalid" OR system.auth.ssh.event: "Failed")

# Successful SSH logins
agent.name: "ubuntutarget" AND
event.dataset: "system.auth" AND
system.auth.ssh.event: "Accepted"

```
# SSH Brute Force Detection Alert
```kql
# Verify alert fired — check Security Alerts index
kibana.alert.rule.name: "SSH Brute Force Activity - ubuntutarget"
```

# RDP Brute Force Detection  Queries
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

# Key Log Queries (C2 Attack Chain)

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