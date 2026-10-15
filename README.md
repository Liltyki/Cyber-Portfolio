# 👋 Hey, I'm Yann Danhier

> 🛡️ **Junior SOC Analyst** · Blue Team · CompTIA Security+

> 📍 Based in Lille, France · Open to opportunities across Europe · Available now

---


I moved into cybersecurity in December 2025, and I learn by building. This portfolio documents that work: a home lab I designed and secured myself, a full phishing analysis project, my own investigation methodology, and write-ups from hands-on investigations.

The goal isn't to show that I can solve challenges. It's to show that I can think like a SOC analyst: investigate, separate what the evidence proves from what it suggests, document, and recommend.

---

## 📂 What's in this portfolio

| Section | What you'll find |
| :--- | :--- |
| 🎣 [**Phishing**](Phishing/) | A complete project: building an isolated phishing lab, then analysing the emails like a reported incident (header analysis, homoglyph domain confirmed at byte level, IOCs, MITRE mapping) |
| 🧭 [**Methodology**](Methodology/) | My own playbooks: how I start an investigation (5 W's, triage, correlation, 20 alert types) and the matching SPL / KQL queries |
| 🔬 [**Lab**](Lab/) | How I built and hardened my home lab, and the security decisions behind it |
| 📝 [**Write-ups**](Writeups/) | TryHackMe investigations rebuilding full attack chains, each mapped to MITRE ATT&CK |

---

## 🔬 Home lab

A self-managed VPS (OVH), designed so that **nothing is exposed publicly**.

- Hardened SSH: key-based authentication, non-standard port, Fail2ban, UFW
- WireGuard VPN as the only way in, with every service bound to the VPN interface
- Docker for service isolation
- Gophish + Mailpit for phishing simulation and telemetry generation
- DVWA and a Kali container for attack/defence scenarios
- 🔄 SIEM deployment in progress, to centralise logs and run detection scenarios

One lesson from building it: **Docker bypasses UFW**. Published container ports are handled before UFW's rules, so firewall rules on those ports are silently ignored. Binding services to the right interface was the reliable fix. It's documented in the [Lab](Lab/) and [Phishing](Phishing/) sections.

---

## 📝 Featured write-ups

| Investigation | Focus |
| :--- | :--- |
| **Analysing a Phishing Email** | Header analysis, Cyrillic homoglyph sender domain, IOC extraction |
| **Slingshot** | Web server compromise in Elastic: recon, brute force, web shell, LFI, database tampering |
| **Tempest** | Full incident response chain, PrintSpoofer privilege escalation, persistence |
| **Benign** | Splunk, Windows Event ID 4688, LOLBin detection |
| **Boogeyman** | Phishing, network and PowerShell history analysis |
| **Health Hazard** | Malicious package, encoded PowerShell, persistence |

---

## 🛠️ Skills & tools

| Domain | Tools | What I do with them |
| :--- | :--- | :--- |
| **SIEM & detection** | Splunk · Elastic | SPL and Kibana queries, alert triage, log analysis, threat hunting |
| **Endpoint** | Windows Event Logs · Sysmon | Event IDs 4624, 4625, 4688, 4698, 4720, process tree analysis, LOLBins |
| **Network** | Wireshark · tcpdump · TShark | PCAP investigation, traffic analysis, IOC extraction |
| **Email** | MXToolbox · hexdump | Header analysis, sender spoofing, homoglyph detection, phishing triage |
| **Threat intelligence** | MITRE ATT&CK · VirusTotal · urlscan.io · CyberChef | MITRE mapping, IOC enrichment, URL and domain reputation |
| **Infrastructure** | Linux · Docker · WireGuard · UFW · Fail2ban · Bash | Server hardening, service isolation, VPN segmentation |
| **Offensive basics** | Kali · Nmap · Burp Suite · Gophish | Reproducing attacks to understand what I need to detect |

---

## 🎓 Certifications & training

- ✅ **CompTIA Security+** · July 2026
- 🔄 **CompTIA CySA+** · exam scheduled November 2026
- ✅ **TryHackMe SEC1** (Cyber Security 101)
- ✅ TryHackMe paths: SOC Level 1 · Jr Penetration Tester · Web Fundamentals · Pre-Security
- 🔄 TryHackMe SOC Level 2 · in progress

---

## 🎯 What I'm looking for

My first **SOC Analyst (L1)** role. I'm comfortable with 24/7 shift work, including nights and weekends, after nearly four years managing a retail store on early, late and weekend shifts. I'm available immediately, on-site, and ready to relocate across Europe.

---

## 📫 Get in touch

📧 yann-dh@orange.fr
💼 [LinkedIn](https://www.linkedin.com/in/yann-danhier/)
🎓 [TryHackMe](https://tryhackme.com/p/LilTyki)
