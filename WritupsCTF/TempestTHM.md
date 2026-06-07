# 🔍 TryHackMe — Tempest | SOC Investigation Writeup

> **Platform**: TryHackMe<br>
> **Room**: Tempest<br>
> **Focus**: Sysmon log analysis

---

## 🎯 Scenario

**Tempest Incident**

In this incident, I act as an Incident Responder following an 
alert triaged by a SOC analyst. The alert was confirmed as 
**CRITICAL severity** and requires further investigation.

According to the analyst, the intrusion started from a malicious 
document. Key information from the alert:

- The malicious document has a `.doc` extension
- The user downloaded the malicious document via `chrome.exe`
- The malicious document executed a chain of commands to achieve 
  code execution

---

## 🔎 Initial Overview

In this TryHackMe room, the platform guides us step by step 
through the investigation. Each question and answer progressively 
builds our understanding of the incident.

---

## 🚪 Initial Access — Malicious Document

The alert gives us a few pieces of information: the malicious 
`.doc` file and the chain of commands executed by that document.

I need to find the file and its associated PID to start my 
investigation. Before going through every log, I need to convert 
the `sysmon.evtx` into a CSV file using **EvtxECmd**, then 
analyze it with **Timeline Explorer**.

<img width="700" alt="EvtxECmd conversion" src="https://github.com/user-attachments/assets/8214c268-b7bf-4038-92e2-7a2469965ba8" />

The room gives us some clues to start the investigation: 
**"Follow the child processes of WinWord.exe"**. By tracking 
WinWord.exe, we discover the malicious `.doc`, its PID, and the 
username.

WinWord.exe is the **parent process** of `free_magicules.doc`,
that's how I identify the malicious document. I can also filter 
with **Event ID 11 (File Create)** for correlation.

<img width="800" alt="Process creation" src="https://github.com/user-attachments/assets/fb4655c8-9386-4573-b591-c50c65a90f65" />

<img width="700" alt="File create event" src="https://github.com/user-attachments/assets/a3f3c00f-5750-4f6c-9d44-36f90cf51cbf" />

I need to continue the investigation and understand how the 
attacker downloaded `free_magicules.doc`. My plan is to filter 
with **Event ID 22 (DNS query)** and PID 496.

<img width="600" alt="DNS query" src="https://github.com/user-attachments/assets/963ecc34-934c-47ea-9ea8-92ef3feb35e5" />

I got the IP. Now I continue the investigation with PID 496 as 
ParentProcessId to find what happened next and I discovered 
the payload.

<img width="750" alt="Payload discovery" src="https://github.com/user-attachments/assets/588c8dcf-facc-4f11-86f9-50cfbc91a01b" />

I used **CyberChef** to decode the Base64 payload.

<img width="700" alt="CyberChef Base64 decode" src="https://github.com/user-attachments/assets/e7b11011-33f7-4280-8a49-1b74c4f41012" />
