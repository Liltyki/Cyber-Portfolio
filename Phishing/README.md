# 🎣 Phishing Analysis — Lab, Investigation & Detection
 
> Building an isolated environment to generate phishing telemetry, then analysing and detecting it from the defender's side.
 
Phishing remains the most common initial access vector, and triaging reported emails is one of the most frequent tasks on a SOC Tier 1 desk. Rather than only reading about it, I built a controlled environment where I can generate realistic phishing artefacts, then investigate them 
 
This project covers the full loop: **infrastructure → generation → analysis → detection.**
 
---
 
## 🏗️ Lab architecture
 
The lab runs on a personal VPS, with every service bound to a WireGuard interface rather than exposed publicly.
 
```
                    VPS (Ubuntu)
    ┌──────────────────────────────────────────────┐
    │                                              │
    │   [ Gophish ]              [ Mailpit ]       │
    │    admin  10.8.0.1:3333     SMTP  127.0.0.1  │
    │    phish  10.8.0.1:8081     UI    10.8.0.1   │
    │         │                                    │
    │         └──► events ──► gophish.db           │
    │                                              │
    └──────────────────────────────────────────────┘
              ▲
              │  WireGuard tunnel — the only way in
              │
         [ Analyst workstation ]
```
 
## How I set it up, and why

Nothing here listens on the public IP. Every service is bound to the WireGuard interface, so the only way to reach the lab is through the VPN. I could have left the ports open and filtered them with firewall rules, but binding to the right interface felt safer: there is no rule to forget or get around, the service simply does not exist anywhere else.

The SMTP port goes one step further and sits on 127.0.0.1. Gophish runs on the same box, so it has no reason to be reachable at all, not even over the VPN.

Using a mail sink instead of a real mail server was the other deliberate choice. Messages are captured locally and physically cannot reach a real inbox. I'd rather the lab be safe because of how it's built than because I remembered to be careful.

---
 
## 📄 Articles
 
| # | Article | Focus |
|---|---------|-------|
| 01 | [Building the lab](01-lab-setup.md) | Isolated infrastructure, interface binding, network segmentation, the Docker/UFW pitfall |
| 02 | [Analysing a phishing email](02-email-analysis.md) | Header analysis, sender discrepancies, IOC extraction, MITRE mapping |
| 03 | [Detecting phishing campaigns](03-detection.md) | Ingesting campaign telemetry, writing detection logic, tuning false positives |
 
---
 
## 🔬 Investigation methodology
 
The analysis in this project follows a consistent structure, which I apply to any reported email:
 
**1. Headers**
Comparing `From` against `Return-Path` and `Envelope-From` to surface spoofing. Reading the `Received:` chain bottom-up to reconstruct the real delivery path. Checking `Authentication-Results` for SPF, DKIM and DMARC verdicts. Looking at `Message-ID` consistency and `X-Mailer` for tooling clues.
 
**2. Body **
Identifying the pretext: urgency, authority, fear, or curiosity. This matters less for detection than for awareness training, but it shapes the recommendation.
 
**3. URLs and attachments **
Extracting links, comparing displayed text against the real target, and analysing destinations through sandboxed services rather than a browser.
 
**4. IOC extraction**
Domains, IPs, hashes and URLs, always defanged (`hxxp://domain[.]com`) so they cannot be clicked from the report.
 
**5. Verdict, response and mapping**
A clear conclusion, concrete actions (block, reset, educate), and mapping to MITRE ATT&CK.
 
---
 
## 🎯 MITRE ATT&CK coverage
 
| Tactic | Technique | ID |
|---|---|---|
| Initial Access | Phishing | T1566 |
| Initial Access | Phishing: Spearphishing Link | T1566.002 |
| Reconnaissance | Gather Victim Identity Information | T1589 |
| Credential Access | Input Capture: Web Portal Capture | T1056.003 |
| Collection | Automated Collection (tracking pixel) | T1119 |
 
---
 
## ⚠️ Known limitations
 
Being explicit about what a lab cannot demonstrate matters as much as what it can.
 
**SPF / DKIM / DMARC results are absent by design.** These are evaluated by the *receiving* mail server against DNS records. A local mail sink performs no such validation, and `.test` domains carry no DNS entries, so no Authentication-Result header is produced. Article 02 therefore covers these mechanisms against a real-world sample rather than lab-generated mail.
 
**Open rates are unreliable.** Tracking pixels only fire when a real mail client fetches remote images. This mirrors production reality, where most clients block remote content by default and open rates are systematically underestimated.
 
---
 
## 🧰 Tools
 
`Gophish` · `Mailpit` · `Docker` · `WireGuard` · `CyberChef` · `urlscan.io` · `tcpdump` / `TShark`


## 📫 Contact
 
🎓 [TryHackMe](https://tryhackme.com/p/LilTyki) · 💼 [LinkedIn](https://www.linkedin.com/in/yann-danhier/)

 
