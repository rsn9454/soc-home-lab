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