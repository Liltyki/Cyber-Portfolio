# From Alert to Query — Splunk (SPL) & Sentinel (KQL) for 20 Alert Types

## Introduction

In my previous article, [*How to Start an investigation.md*](#), I shared the **method** I use to start an investigation, the 5 W, the triage, and correlation. This second article is the tooling side of that playbook: for each of the 20 alert types, a query in **SPL** (Splunk) and its equivalent in **KQL** (Microsoft Sentinel / Defender). I often use Splunk, i decided to add KQL (sentinel) with IA for future investigation if i have to work with sentinel in a future.

## Three queries, explained

Before dumping the full table, here are three queries broken down, because copying a query is easy; understanding *why* it works is the actual skill.

### 1 — Repeated failed SSH logins (brute force)

**SPL:**
```spl
index=linux sourcetype=linux_secure "Failed password"
| stats count by src_ip, user
| where count > 20
| sort -count
```

- `index=linux sourcetype=linux_secure "Failed password"` —> pull only Linux auth events that contain a failed password.
- `stats count by src_ip, user` —> group them: how many failures per source IP and per targeted user?
- `where count > 20` —> keep only the noisy ones (a handful of failures is normal; 20+ is suspicious).
- `sort -count` —> worst offenders first.


### 2 — PowerShell with an encoded command

**SPL:**
```spl
index=windows (EventCode=4688 OR EventCode=4104)
(powershell AND ("-enc" OR "-EncodedCommand"))
| table _time, host, User, ParentImage, CommandLine
```

Here the key is not just *finding* the encoded PowerShell, but showing the `ParentImage` what launched it. PowerShell launched by a script is normal; PowerShell launched by `winword.exe` is not.

### 3 — Port scan

**SPL:**
```spl
index=firewall action=blocked
| stats dc(dest_port) as ports by src_ip
| where ports > 100
| sort -ports
```

The trick is `dc(dest_port)` a **distinct count** of destination ports per source IP. One IP touching 100+ different ports in a short window is the signature of a scan, not normal traffic.

## The 20 alert types — SPL & KQL

Same order as the method article. Adapt field names to your environment.

### 1. Repeated failed SSH logins — `T1110`
```spl
index=linux sourcetype=linux_secure "Failed password"
| stats count by src_ip, user | where count > 20 | sort -count
```
```kql
Syslog
| where SyslogMessage has "Failed password"
| extend src = extract("from ([0-9.]+)", 1, SyslogMessage)
| summarize Attempts = count() by src, HostName | where Attempts > 20
```

### 2. Successful login after brute force — `T1110`
```spl
index=linux sourcetype=linux_secure ("Failed password" OR "Accepted password")
| transaction src_ip maxspan=10m
| search "Failed password" "Accepted password"
```
```kql
Syslog
| where SyslogMessage has_any ("Failed password", "Accepted password")
| extend src = extract("from ([0-9.]+)", 1, SyslogMessage)
| summarize Fails = countif(SyslogMessage has "Failed"),
            Ok = countif(SyslogMessage has "Accepted") by src
| where Fails > 10 and Ok > 0
```

### 3. Login from an unusual country — `T1078`
```spl
index=auth action=success
| iplocation src_ip
| stats dc(Country) as countries values(Country) by user
| where countries > 1
```
```kql
SigninLogs
| where ResultType == 0
| summarize Countries = dcount(Location), make_set(Location) by UserPrincipalName
| where Countries > 1
```

### 4. Login outside business hours — `T1078`
```spl
index=auth action=success
| eval hour=strftime(_time,"%H")
| where hour < 6 OR hour > 22
| stats count by user, host, hour
```
```kql
SigninLogs
| where ResultType == 0
| extend h = datetime_part("hour", TimeGenerated)
| where h < 6 or h > 22
| project TimeGenerated, UserPrincipalName, IPAddress, h
```

### 5. PowerShell encoded command (`-enc`) — `T1059.001`
```spl
index=windows (EventCode=4688 OR EventCode=4104)
(powershell AND ("-enc" OR "-EncodedCommand"))
| table _time, host, User, ParentImage, CommandLine
```
```kql
DeviceProcessEvents
| where FileName =~ "powershell.exe"
| where ProcessCommandLine has_any ("-enc", "-EncodedCommand")
| project Timestamp, DeviceName, AccountName, ProcessCommandLine
```

### 6. Office spawns cmd/PowerShell — `T1566.001`
```spl
index=windows EventCode=4688
(ParentImage="*winword.exe" OR ParentImage="*excel.exe")
(New_Process_Name="*cmd.exe" OR New_Process_Name="*powershell.exe")
```
```kql
DeviceProcessEvents
| where InitiatingProcessFileName in~ ("winword.exe", "excel.exe")
| where FileName in~ ("cmd.exe", "powershell.exe")
| project Timestamp, DeviceName, InitiatingProcessFileName, ProcessCommandLine
```

### 7. Port scan detected — `T1046`
```spl
index=firewall action=blocked
| stats dc(dest_port) as ports by src_ip | where ports > 100 | sort -ports
```
```kql
CommonSecurityLog
| summarize Ports = dcount(DestinationPort) by SourceIP
| where Ports > 100 | sort by Ports desc
```

### 8. SQLi payload in web logs — `T1190`
```spl
index=web (uri_query="*union*" OR uri_query="*select*" OR uri_query="*'--*")
| stats count by src_ip, uri_path, status
```
```kql
W3CIISLog
| where csUriQuery has_any ("union", "select", "--", "' or")
| project TimeGenerated, cIP, csUriStem, csUriQuery, scStatus
```

### 9. Malware detected / quarantined — `T1204`
```spl
index=av (action=quarantine OR action=blocked)
| table _time, host, signature, file_path, user
```
```kql
DeviceEvents
| where ActionType startswith "Antivirus"
| project Timestamp, DeviceName, FileName, FolderPath, InitiatingProcessAccountName
```

### 10. Traffic to a C2 domain/IP — `T1071`
```spl
index=proxy
| lookup threat_intel_domains domain OUTPUT malicious
| where malicious=1 | stats count by src_ip, domain
```
```kql
DeviceNetworkEvents
| join kind=inner (ThreatIntelligenceIndicator | where Active == true)
    on $left.RemoteUrl == $right.DomainName
| project Timestamp, DeviceName, RemoteUrl, InitiatingProcessFileName
```

### 11. Exfiltration: large outbound volume — `T1048`
```spl
index=firewall direction=outbound
| stats sum(bytes_out) as total by src_ip, dest_ip
| where total > 1000000000 | sort -total
```
```kql
CommonSecurityLog
| summarize Sent = sum(SentBytes) by SourceIP, DestinationIP
| where Sent > 1000000000 | sort by Sent desc
```

### 12. New admin account created — `T1136`
```spl
index=windows EventCode=4720
| table _time, host, Subject_Account, Target_Account
```
```kql
SecurityEvent
| where EventID == 4720
| project TimeGenerated, Computer, SubjectAccount, TargetAccount
```

### 13. Added to a privileged group — `T1098`
```spl
index=windows (EventCode=4728 OR EventCode=4732 OR EventCode=4756)
Group_Name="*Admins*"
| table _time, host, Subject_Account, Member_Name, Group_Name
```
```kql
SecurityEvent
| where EventID in (4728, 4732, 4756)
| where GroupName has "Admins"
| project TimeGenerated, Computer, SubjectAccount, MemberName, GroupName
```

### 14. Event logs cleared — `T1070.001`
```spl
index=windows (EventCode=1102 OR EventCode=104)
| table _time, host, User
```
```kql
SecurityEvent
| where EventID == 1102
| project TimeGenerated, Computer, SubjectUserName, Activity
```

### 15. Antivirus / Defender disabled — `T1562.001`
```spl
index=windows ("DisableRealtimeMonitoring" OR EventCode=5001)
| table _time, host, User, CommandLine
```
```kql
DeviceRegistryEvents
| where RegistryKey has "Windows Defender"
| where RegistryValueName == "DisableAntiSpyware" and RegistryValueData == "1"
| project Timestamp, DeviceName, InitiatingProcessAccountName
```

### 16. lsass.exe access / Mimikatz — `T1003.001`
```spl
index=windows EventCode=10 TargetImage="*lsass.exe"
| stats count by host, SourceImage, GrantedAccess
```
```kql
DeviceProcessEvents
| where ProcessCommandLine has_any ("sekurlsa", "lsadump", "mimikatz")
| project Timestamp, DeviceName, AccountName, ProcessCommandLine
```

### 17. Reported phishing email — `T1566`
```spl
index=email (subject="*[REPORTED]*" OR verdict=phish)
| table _time, sender, recipient, subject, url
```
```kql
EmailEvents
| where ThreatTypes has "Phish"
| project Timestamp, SenderFromAddress, RecipientEmailAddress, Subject, Url
```

### 18. USB media inserted — `T1091`
```spl
index=windows EventCode=6416
| table _time, host, User, Device_Description
```
```kql
DeviceEvents
| where ActionType == "UsbDriveMounted"
| project Timestamp, DeviceName, InitiatingProcessAccountName, AdditionalFields
```

### 19. Scheduled task / service created — `T1053`
```spl
index=windows (EventCode=4698 OR EventCode=7045)
| table _time, host, User, Task_Name, Service_Name
```
```kql
SecurityEvent
| where EventID in (4698, 7045)
| project TimeGenerated, Computer, SubjectAccount, ServiceName
```

### 20. Abnormal DNS spike (tunneling) — `T1071.004`
```spl
index=dns
| eval qlen=len(query)
| where qlen > 50 | stats count by src_ip, query | sort -count
```
```kql
DnsEvents
| extend qlen = strlen(Name)
| where qlen > 50
| summarize Requests = count() by ClientIP, Name | sort by Requests desc
```

## How to adapt these to your environment

None of these will work by copy-paste  and that's normal. Before running any of them, check:

- Index / source names. `index=windows` on my notes might be `index=wineventlog` on other instance. 
- Field names. `src_ip`, `EventCode`, `dest_port` depend on how your data is parsed (CIM in Splunk, the schema of each table in Sentinel).
- Thresholds. `count > 20`, `ports > 100` are starting points, not truths. Tune them to your baseline to reduce false positives.

## Conclusion

A method tells you where to look; these queries are how you ask. Together with the previous article, this is the small playbook I keep on my desk to move faster without skipping steps.

Next step for me: run these against real data. I'm deploying Wazuh in my home lab to centralize logs from Linux, web applications, and a Windows endpoint  so I can replace "template" with "tested."

## 📫 Connect

- 🎓 **TryHackMe:** https://tryhackme.com/p/LilTyki
- 💼 **LinkedIn:** https://www.linkedin.com/in/yann-danhier/
