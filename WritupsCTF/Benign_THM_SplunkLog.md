# 🔍 TryHackMe — Benign | SOC Investigation Writeup

> **Platform**: TryHackMe
> **Room**: Benign
> **Analyse Splunk logs
---

## 🎯 Scenario
One of the client’s indicated a potentially suspicious process execution indicating one of the hosts from the HR department was compromised. Some tools related to network information gathering / scheduled tasks were executed which confirmed the suspicion. Due to limited resources, we could only pull the process execution logs with Event ID: 4688 and ingested them into with the index win_eventlogs for further investigation.

About the Network Information :The network is divided into three logical segments. It will help in the investigation.
IT Department
    James
    Moin
    Katrina
HR department
    Haroon
    Chris
    Diana
Marketing department
    Bell
    Amelia
    Deepak

### 

More than just answer every question like a robot i need to undsertand what's happens in these splunk logs. In this scenario i got a alert as a SOC for a potentially suspicious process execution from the HR department and one suspicious Scheduled ta
