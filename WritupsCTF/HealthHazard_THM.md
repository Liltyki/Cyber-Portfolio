# 🔍 TryHackMe — Health Hazard | Threat Hunting Write-up

> **Platform:**: TryHackMe<br>
> **Room:**: Health Hazard<br>
> **Focus:**: Splunk · Windows logs<br>
> 

---

## 📜 Scenario

**Issued by:** TryDetectThis Intelligence
**Classification:** Internal – TLP:AMBER

TryDetectThis Intelligence has identified a coordinated supply chain attack campaign targeting open-source ecosystems, specifically npm and Python package repositories. The campaign appears to be orchestrated by a threat actor leveraging long-term infiltration of neglected or low-profile projects to weaponize legitimate packages.

The attacker's strategy involves contributing to moderately used but under-maintained libraries, gaining contributor or maintainer status through helpful commits. Once trusted, they publish malicious updates, embedding post-installation payloads or obfuscated backdoors within version releases that appear minor or maintenance-related.

These weaponized libraries often act as stagers for follow-on actions — such as downloading secondary payloads, establishing persistence, or exfiltrating tokens and credentials from developer machines. Due to their presence in tutorials, starter templates, or widely shared codebases, they have a high chance of spreading through organic adoption.

**Hypothesis:** 

An attacker may have leveraged a compromised third-party software package to gain initial access to the system and silently stage a payload for later execution. They likely established persistence to maintain access without immediate detection.

---

## 🔎 Initial Overview

From the scenario, I understood this was most likely a supply-chain attack through the open-source npm ecosystem. The goal of this room is to uncover the three stages of the attack chain.

My first instinct was to start from npm inside Splunk and follow the thread from there.

---

## 🚪 Into the Splunk logs

I began by filtering on `npm` in Splunk. Lucky! only 5 events matched.

<img width="720" height="348" alt="Splunk 1" src="https://github.com/user-attachments/assets/0c4060a2-0631-432e-b08a-bbc7fe456ce0" />

I started to watch the `CommandLine` field, because we know the attack chain starts with npm — and the attacker then deploys a payload, establishes persistence, and so on. So the `CommandLine` field is the right place to look.

<img width="533" height="894" alt="Splunk - CommandLine field" src="https://github.com/user-attachments/assets/96886038-654e-43b1-8d0a-494436f5cbf3" />

And there it was: an encoded command. The trailing `=` was a strong hint that this is Base64, so i dropped it into my best friend CyberChef to decode it.

<img width="1532" height="843" alt="CyberChef - decoded payload" src="https://github.com/user-attachments/assets/03067313-cde0-4216-ab5a-2160913275b9" />

The decoded payload revealed the attacker's intent:

- **Downloads** a malware from a suspicious domain: `wlndows[.]thm` (a deliberate **typosquatting**  an `l` in place of the `i` in *windows* ,designed to mimic the legitimate `windows.com`).
- **Establishes persistence** by writing a Run key at `HKCU\Software\Microsoft\Windows\CurrentVersion\Run` under the value name `Windows Update Monitor`, ensuring the payload re-executes at every user logon.

The Splunk event also gave us the supporting context needed for the incident report: computer name, timeline, Process ID...

At this stage, as a SOC Level 1 analyst, the job is to gather and document these artefacts so the case can be escalated to L2/L3 for deeper response.

---

## ⛓️ Attack Chain & Conclusion

We now have the full picture of the attack chain, from initial access to the persistence method.

```
Compromised npm package (postinstall script)
        │  T1195.002 — Supply Chain Compromise
        ▼
Encoded PowerShell execution (CommandLine)
        │  T1059.001 — PowerShell  |  T1027 — Obfuscation
        ▼
Secondary payload download from wlndows[.]thm
        │  T1105 — Ingress Tool Transfer  |  T1036 — Masquerading (typosquat)
        ▼
Persistence via HKCU\...\Run  ("Windows Update Monitor")
        │  T1547.001 — Registry Run Key
        ▼
Payload re-executes at every logon
```

With the full chain mapped, we can write the report and finish the room.

<img width="1486" height="274" alt="Attack chain stages" src="https://github.com/user-attachments/assets/6ce5f9f0-37bf-4941-9419-59c3f9963a88" />

---

## 🎯 MITRE ATT&CK Mapping

| Tactic | Technique | ID |
|---|---|---|
| Initial Access | Supply Chain Compromise: Software | T1195.002 |
| Execution | Command and Scripting Interpreter: PowerShell | T1059.001 |
| Defense Evasion | Obfuscated Files or Information | T1027 |
| Defense Evasion | Masquerading (typosquatted domain) | T1036 |
| Command & Control | Ingress Tool Transfer | T1105 |
| Persistence | Boot or Logon Autostart Execution: Registry Run Keys | T1547.001 |

---

## 💭 What I Learned

This room was pretty interesting and fast because, compared to other rooms like **Tempest** or **Benign**, we had a real alert and needed to investigate like a SOC Level 1 analyst escalating to SOC Level 2. We don't often write reports in TryHackMe rooms, so this was a good exercise!

---

## 📫 Connect

- 🎓 **TryHackMe:** https://tryhackme.com/p/LilTyki
- 💼 **LinkedIn:** https://www.linkedin.com/in/yann-danhier/
