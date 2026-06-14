# 🔍 TryHackMe — TShark Challenge 2: Teamwork | SOC Investigation Writeup

> **Platform:** TryHackMe
> **Room:** TShark Challenge 2 — Teamwork
> **Focus:** Network traffic analysis with TShark (CLI)

---

## 🎯 Scenario

An alert was triggered: the threat research team discovered a suspicious domain that could be a potential threat to the organization. We're given only that lead and a `.pcap` file on the Desktop.

The goal of this room is to practice our TShark skills  and that's perfect, because we often lean on the GUI (Wireshark) and forget that command-line analysis matters just as much. A solid analyst is comfortable with both.

---

## Identifying the Malicious Domain

First, I need an overview of the web traffic to spot the malicious domain. My first instinct is to look at the HTTP GET requests:

```bash
tshark -r teamwork.pcap -Y "http.request.method == GET"
```

<img width="700" alt="HTTP GET requests in the capture" src="https://github.com/user-attachments/assets/ff4e750d-0afc-43d9-9956-067a5b275dc9" />

This reveals multiple requests to an external IP address, `184.154.127.226`, fetching files such as `/suspecious.php`, `/js/cc.js`, and PayPal-related assets (`paypalsansmallmedium.woff2`).

In a work environment, it isn't normal to find PayPal assets and a credit card validator. At this point, it could be an employee who just bought a new computer on Amazon, or a malicious attacker! I need to keep investigating: find the URL and check it with a tool like VirusTotal.

```bash
tshark -r teamwork.pcap -Y "http.request" -T fields -e http.request.full_uri
```

<img width="700" alt="Full request URIs extracted with TShark" src="https://github.com/user-attachments/assets/dbe25d59-ec23-4062-a9fa-afe3b60b0bf2" />

Bingo, got it! The full URL is now visible.

## Domain Investigation

VirusTotal is an excellent database and threat-hunting tool: it aggregates intelligence from dozens of vendors on hashes, domains and URLs. It's the go-to place to cross-check a finding against known threat data.

<img width="700" alt="VirusTotal lookup of the suspicious domain" src="https://github.com/user-attachments/assets/e225d32d-f8cd-42df-85b7-f95a286f021f" />

For this room, VirusTotal doesn't return much: only a couple of vendors flag the domain. That's likely because the scenario is an educational, controlled case rather than a live campaign. Still, the domain is clearly impersonating PayPal for malicious purposes. Next, let's find the email address tied to this phishing activity.

#### The Email Address

The victim's email address is present in the capture. The following filter dumps the full packet details and extracts every email address with a regex:

```bash
tshark -r teamwork.pcap -V | grep -Eo '[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}'
```

<img width="700" alt="Email address found in the capture" src="https://github.com/user-attachments/assets/4614d806-b5bc-4964-8ddd-b3193afbcd01" />

This is the victim's email address.
And that's it, the scenario is finished! I have all the information I need, and I have the attack timeline. And the answer to every question !


## 🔗 Attack Timeline

1. **Access / Delivery** — The victim's browser reaches `http://184.154.127.226/suspecious.php`, a phishing page impersonating PayPal. *(observed: HTTP GET)*
2. **Fake page rendering** — The browser loads the spoofed PayPal assets (fonts, images) and `cc.js`. *(observed: HTTP GET)*
3. **Credential / card capture** — `cc.js` is likely designed to capture the victim's card data. *(to confirm by inspecting the HTTP POST request)*

**MITRE ATT&CK:** T1566 — Phishing.

## ✅ What I Learned & Conclusion

Starting from a single lead — a suspicious domain — TShark alone was enough to reconstruct the essentials of this incident:

- Internal traffic was reaching an external host (`184.154.127.226`) serving a fake PayPal page — a classic credit-card phishing setup.
- VirusTotal confirmed the domain as known-bad.
- I identified the victim's email address.

I really enjoyed this room: I usually reach for Wireshark instead of TShark, so it was the perfect exercise to improve that skill. I'm also going to install TShark in my own lab for future simulations.

----

### 📫 Get in Touch


🎓 TryHackMe: https://tryhackme.com/p/LilTyki

💼 LinkedIn: https://www.linkedin.com/in/yann-danhier/

📧 Email : yann-dh@orange.fr



