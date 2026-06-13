# 🔍 TryHackMe — Tshark Challenge 1: Teamwork | SOC Investigation Writeup

> **Platform**: TryHackMe<br>
> **Room**: Tsharl Challenge 1 : teamwork <br>
> **Focus**: Tshark Analysis network log

---

## 🎯 Scenario
An alert was triggered: the threat research team discovered a suspicious domain that could be a potential threat to the organization. We're given only that lead and a .pcap file on the Desktop.
The goal of this room is to use our TShark skills and that's perfect, because we often lean on the GUI (Wireshark) and forget that command-line analysis matters just as much. 
---

### Identifying the malicious Domain 


First, I need an overview of the web traffic to spot the malicious domain. My first instinct is to look at the HTTP GET requests:

```bash
  tshark -r teamwork.pcap -Y "http.request.method == GET"
```

<img width="700" alt="Capture d&#39;écran 2026-06-13 173337" src="https://github.com/user-attachments/assets/ff4e750d-0afc-43d9-9956-067a5b275dc9" />


This reveals multiple request to an external IP Address 184.154.127.226 fetching files like /suspecious.php, /js/cc.js , and PayPal-Related assets (paypalsansmallmedium.woff2). 



In a work environment, it isn't normal to find PayPal assets and a credit card validator. At this point, it could be an employee who just bought a new computer on Amazon, or a malicious attacker! I need to keep investigating: find the URL and check it with a tool like VirusTotal.

```bash
tshark -r teamwork.pcap -Y "http.request" -T fields -e http.request.full_uri
```

<img width="700"  alt="Capture d&#39;écran 2026-06-13 181919" src="https://github.com/user-attachments/assets/dbe25d59-ec23-4062-a9fa-afe3b60b0bf2" />

Bingo  go it !the full url is now visible .

## Domain Investigation
VirusTotal is an excellent database and threat-hunting tool: it aggregates intelligence from dozens of vendors on hashes, domains and URLs. It's just the go to place to cross-check a finding against known threat data.


<img width="846" height="741" alt="Capture d&#39;écran 2026-06-13 182237" src="https://github.com/user-attachments/assets/e225d32d-f8cd-42df-85b7-f95a286f021f" />


For this room, VirusTotal doesn't return much: only a couple of vendors flag the domain beetwenn 2017 and 2026. That's likely because the scenario is an educational, controlled case rather than a live campaign. Still, the domain is clearly impersonating PayPal for malicious purposes. Next, let's find the email address tied to this phishing activity.


#### The Email Address

The victim's email address is present in the capture. The following filter lets us extract it from the traffic:

```bash
tshark -r teamwork.pcap -V | grep -Eo '[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}' 
```

<img width="700" alt="Capture d&#39;écran 2026-06-13 182659" src="https://github.com/user-attachments/assets/4614d806-b5bc-4964-8ddd-b3193afbcd01" />

This is the Victim email adress! 


## ✅ What i learned and Conclusion

Starting from a single lead (a suspicious domain) Tshark alone was enough to reconstruct the essentials of this incident 
  -Internal traffic was reaching an external Host 184.154.127.226 serving a fake paypal page -> A classic credit-card phishing setup
  -Virus total confirmed the domain as know bad
  -I identified the victim's email addres .

And in my point of view i really enjoy that room because i often use wireshark instead of tshark and that was the perfect exercice for me to improve my skill! And also i will install Tshark on my own lab for my future simulation ! 

##🔗 Attack Timeline

1- Access / Delivery — The victim's browser reaches http://184.154.127.226/suspecious.php, a phishing page impersonating PayPal. (observed: HTTP GET)
2- Fake page rendering — The browser loads the spoofed PayPal assets (fonts, images) and cc.js. (observed: HTTP GET)
3- Credential / card capture — cc.js is likely designed to capture the victim's card data. (to confirm by inspecting the HTTP POST request)

MITRE ATT&CK: T1566 — Phishing.



📫 Get in Touch

🎓 TryHackMe: https://tryhackme.com/p/LilTyki

💼 LinkedIn: https://www.linkedin.com/in/yann-danhier/

📧 Email : yann-dh@orange.fr



































