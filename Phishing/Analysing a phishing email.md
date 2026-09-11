# 🔍 Analysing a Phishing Email

As I said in article 1, I used my own home lab to create a phishing campaign and analyse the resulting emails. I asked an AI assistant to generate realistic phishing content, the first one being the classic "you need to reset your password". 

## The Scenarios 

Working from a scenario is more engaging than analysing raw email data. Let me introduce ChocoCompany, the world's leading chocolate manufacturer. Their accounting department has just alerted the security team about two suspicious emails.
The two members of the accounting team received that mail.


<img width="400" height="400" alt="Capture d&#39;écran 2026-09-11 185829" src="https://github.com/user-attachments/assets/a41d733b-cbd8-47f9-90e5-f11e5071e386" />

ChocoCompany is fortunate to have employees trained to recognise phishing attempts. Analysing a suspicious email answers first :is this actually malicious ? who sent it, and what were they after ? It also produces IOCs that can be reused later.

This article focuses on the email analysis itself. But a SOC analyst doesn't stop at closing the ticket, other questions remain open and neither can be answered from the message alone:

Did anyone else receive it? One report usually means several deliveries.
Did anyone click? That's the difference between a blocked attempt and an active incident.

Both are answered in the logs, not in the Email



## 👀 First impressions


The email looks like a legitimate  password renewal from chococompany.com. They target john and Fiona from accounting department and it comes from a mail address that looks legitimate. This is a spear phishing:
A phishing compaign to target chococompany, the content look exactly like a real mail from ChocoCompagny, the sender's address  look like a real mail from it-support@choco.com and they know John and Fiona's informations (email , name ...) 

To determine if the mail is suspicious or not  I need to analyse header ! 


 ## 📬 Header analysis

To analysis email's header I will use mxtoolbox.com , i will copy the EML content to the tools and i will gain more information than the simple view of the mail. This scenario is build in my lab so i design this campaign to not be reached by internet . 
So DMARC DKIM and SPF will not be analysed here but in a future article.

<img width="400" height="400" alt="Capture d&#39;écran 2026-09-11 191807" src="https://github.com/user-attachments/assets/e2e31f36-ebce-4510-a6d0-124e51094824" />

Ok the return-Path and the sender are suspicious. They are different :

Return-Path : it-support@choco.com
From : it-support@сh0co.com

As a member of chococompany i know the email from IT-support is : it-support@choco.com. But as i see the mail comes from it-support@ch0co.com . The attacker change the o to a 0 to deceive the victim. 
But sometimes it is not  as obvious as that, and the human's eye cannot spot the difference beetween a'c' in latin alphabet and a 'c' in cyrilic alphabet. hexdump will help us to determine if the 2 email match.

<img width="640" height="100" alt="Capture d&#39;écran 2026-09-11 192444" src="https://github.com/user-attachments/assets/35915eef-25bd-480d-8a97-bb369ec679d4" />


The first character is 0xd1 0x81 -> UTF-8 for U+0441, Cyrillic small letter ES  not the Latin c (0x63). The o in "choco" is also a zero. Two separate substitution techniques in one address, and neither is visible to the naked eye.

This is a homoglyph attack (T1583.001). The displayed domain is not the one being used.

The difference between the Return-PATH and the From is not efficient. Most of companies can have a different return-path to centralize their answers etc.

They are no attachments to analyse so it's time to check the body.

## ✉️ The message body

The EML carries the body message. The message is another thing to analysis after headers and attachments. 


<img width="1000" height="700" alt="Capture d&#39;écran 2026-09-11 212713" src="https://github.com/user-attachments/assets/9fe4bf4a-efef-4bd5-b8dd-2f265932fd1b" />


With the IA era the grammar and spelling are close to be perfect, so poor writing is no longer a reliable indicator. This is a mail for a reset password, and the link point to :

' 
http://ch0coCompagny.com/reset-password?rid=V4Ha0nI

'

Two things stand out. The zero substitution again and Compagny instead of Company. This is a second attacker-controlled domain, distinct from the one used to send the message. Registering separate infrastructure for sending and for hosting is common practice: it limits the damage when one of them gets blocked.



### Handling the URL
 
The rule first: never open a suspicious URL from your own workstation. Submit it to a sandboxed service like urlscan.io or VirusTotal which returns a screenshot, the request chain, the contacted domains and any resolved IP. Those are indicators in their own right, and they cost nothing to collect.
 
Here, neither service returns anything: the domain doesn't exist, because in this simulation I didn't build a fake website to impersonate the company.
 
Where it *is* appropriate to go further  a dedicated VM, isolated from the network, with no credentials on it  curl gives you the response without rendering anything:
 
```bash
curl -i "hxxp://ch0coCompagny[.]com/reset-password?rid=V4Ha0nI"
```
 
That's how you'd map the landing page, collect further IOCs and understand what the campaign was actually serving.



## ⚖️ Verdict and Action :

The conclusion is : The email is a phishing. We can conclude we several  indicator, the Email address, the impersonation website with missspelling and the characters substituion. 
As i said early, we need to determine if these email are the only send to the employee ? and if anyone are already click on it. 
To answer those questions we need to go in the log like network logs for any connection to  the domain or mail logs for other recipients of the campaign.
 
🔴 **Immediate** | Block both domains сh0co[.]com and ch0coCompagny[.]com at the mail gateway and the proxy 

🔴 **Immediate** | Search the mail logs for other recipients of the same campaign 

🟠 **Short term**| Check proxy logs for any connection to the link domain that tells you whether anyone clicked 

🟠 **Short term**| If anyone submitted credentials, reset them and revoke active sessions 

🟡 **Follow-up** | Brief the accounting team specifically. Impersonation of internal IT deserves a targeted reminder, not a generic one 



## 🔎 Indicators of Compromise
 
All indicators are defanged so they cannot be clicked from this report.
 
| Type | Indicator | Notes |
| :--- | :--- | :--- |
| Domain | `сh0co[.]com` | Sender domain : Cyrillic `с` (U+0441) + zero substitution |
| Domain | `ch0coCompagny[.]com` | Link domain : zero substitution + misspelling (`Compagny`) |
| Email | `it-support@сh0co[.]com` | Spoofed sender, impersonating internal IT |
| URL | `hxxp://ch0coCompagny[.]com/reset-password?rid=V4Ha0nI` | Action link |
| URL | `hxxp://ch0coCompagny[.]com/reset-password/track?rid=V4Ha0nI` | Hidden tracking pixel (`display:none`) |
| Header | `X-Mailer: gophish` | Sending tool exposed in clear text |
| Header | `Message-Id: ...@vps-295ed427` | Originating host, unrelated to `choco.com` |

## 🎯 MITRE ATT&CK
 
| Tactic | Technique | ID | Evidence |
| :--- | :--- | :--- | :--- |
| Resource Development | Acquire Infrastructure: Domains | `T1583.001` | Two lookalike domains registered by the attacker |
| Reconnaissance | Gather Victim Identity Information | `T1589` | Correct names and email format for two accounting staff |
| Initial Access | Phishing: Spearphishing Link | `T1566.002` | Targeted message with an embedded malicious link |
| Defense Evasion | Impersonation | `T1656` | Message presents as the internal IT service desk |


> [!IMPORTANT]
> **Suspected, not confirmed.** The pretext points toward credential harvesting (`T1056.003`  Input Capture: Web Portal Capture), but in this simulation I did not build a landing page, so nothing proves what the link actually served. On a real investigation this is exactly where URL analysis would answer the question  and where I'd stop short of writing a conclusion the evidence doesn't support.


## 📫 Connect

- 🎓 **TryHackMe:** https://tryhackme.com/p/LilTyki
- 💼 **LinkedIn:** https://www.linkedin.com/in/yann-danhier/












