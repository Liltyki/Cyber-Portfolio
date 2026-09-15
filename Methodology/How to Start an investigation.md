# How to star an investigation after an alert  ? A soc  analyst's starting point

## Introduction 


When I started learning the SOC analyst role, I had no idea how to start an investigation. When you learn cybersecurity on your own, you don't have a company playbook. You are alone in front of the desk. I learned the tools, the theory, I improved my skills, but the first time I started a TryHackMe room I thought: "OK, I have no idea how to start." After some research, I built a little playbook on my desk to help me memorize everything I needed.
 
A SOC analyst has to decide whether an alert is a false positive or a true positive, and the challenge is to stay fast and rigorous: answering *What? Where? When? Who? Why?* quickly enough to keep up, without rushing so much that a real threat slips through. That balance is exactly why having a method matters.




## The 5 w and the triage 

Before going through Windows logs, network logs, or whatever the source is, I need context about the alert. As I said earlier, answering the 5 W gives me that context:

- What ? Which rule fired, and what exactly does it detect ?
- where ? which hosts and Ips are involved ? 
- when ? what is the timeline ?
- Who ? which user or account is involved ?
- Why ? is this a true or false ? or benign activity ?
  
This context should give me enough information to decide whether the alert is a true or false positive, and whether it needs to be escalated or closed.

A method is also what protects a SOC analyst from fatigue. An analyst can face thousands of alerts per day, and after a few hundred, tiredness sets in and effectiveness drops. That's why building your own little playbook early in your career helps.


## The LOLbins : 

Sometime its easy and very fast to determine if the alert is a true positive or not. Like if a powershell is launch by an excel or contains -enc or IEX in the queries that most of the time a True positive. Understand LOLBins for windows and linux will increase the soc efficiente. Lot of time an attacker will use LOLbins (living off the land Binaries) to avoid anti-virus. A corect set of rule and the behavior user is the keys to alert these kind of exploit. 

Sometimes it's easy and fast to determine whether an alert is a true positive. For example, if PowerShell is launched by Excel, or contains `-enc` or `IEX` in the command line, it's a true positive most of the time. Understanding LOLBins on Windows and Linux increases a SOC analyst's efficiency. Attackers often use LOLBins (Living Off the Land Binaries) to evade antivirus. A correct set of rules combined with user-behavior context is the key to catching this kind of activity.


## Into the logs : 


A single alert rarely tells a story, the real work is correlation. To understand what is behind an alert, we need to correlate different sources. Take an example: it's 9 AM, during working hours. A server gets 5 failed SSH logins from an unknown IP. This could be a remote employee who forgot to connect through the VPN, or an attacker trying to brute-force SSH. To determine which one it is, we need to correlate different pieces of information and sources:

- Check the IP reputation and origin.
- Check the user's history: does this employee usually work at this time? Is anyone working remotely?
- Analyze the attempt pattern: 5 failed attempts followed by a successful login? Or 5 failed attempts across 50 different accounts?

Every question is answered with more information we pick up in the logs. As I said earlier, a few months ago, the first time I investigated an alert, I thought: *"OK, I know I need to check the network logs, the syslog... but how? What's the query? Where do I start?"* That's why I made a table to enhance the process and answer faster.

## The 20 alert types :
 
For each: the alert, the logs to check first,and  where to start.


| # | Alert | Logs to check | Where to start | MITRE |
|---|---|---|---|---|
| 1 | Repeated failed SSH logins | Linux `auth.log` | Count attempts per source IP; is there a success after? | T1110 |
| 2 | Successful login after brute force | Linux `auth.log` | Correlate failures then success on the same IP | T1110 |
| 3 | Login from an unusual country | Auth logs / SigninLogs | Compare to the user's usual geo (impossible travel) | T1078 |
| 4 | Login outside business hours | Auth logs / SigninLogs | Compare to the account's usual hours | T1078 |
| 5 | PowerShell encoded command (`-enc`) | Win 4688 / 4104 / Sysmon 1 | Decode the base64, check parent process | T1059.001 |
| 6 | Office spawns cmd/PowerShell | Win 4688 / Sysmon 1 | Abnormal parent-child chain = malicious macro? | T1566.001 |
| 7 | Port scan detected | Firewall / IDS | Source IP internal/external? How many ports? | T1046 |
| 8 | SQLi payload in web logs | Web `access.log` / WAF | Find the URL/param; 500 = possible success | T1190 |
| 9 | Malware detected / quarantined | AV / EDR | Confirm quarantine, find the vector, scope | T1204 |
| 10 | Traffic to a C2 domain/IP | Proxy / DNS / firewall + TI | Source process? Regular beaconing? Isolate | T1071 |
| 11 | Exfiltration: large outbound volume | Firewall / NetFlow | Which host, destination, volume, duration | T1048 |
| 12 | New admin account created | Win 4720 | Who created it, was it authorized? | T1136 |
| 13 | Added to a privileged group | Win 4728 / 4732 / 4756 | Legitimate? Correlate with source account activity | T1098 |
| 14 | Event logs cleared | Win 1102 / 104 | Highly suspect (anti-forensics): who, what host? | T1070.001 |
| 15 | Antivirus / Defender disabled | Defender 5001 / Registry | Manual or malicious? Correlate with what follows | T1562.001 |
| 16 | lsass.exe access / Mimikatz | Sysmon 10 / EDR | Likely credential theft: isolate host, scope | T1003.001 |
| 17 | Reported phishing email | Mail gateway / O365 | Sender, links/attachments, who else received it | T1566 |
| 18 | USB media inserted | Win 6416 / EDR | Policy respected? Files copied? User context | T1091 |
| 19 | Scheduled task / service created | Win 4698 / 7045 | Persistence? What does it run, created by whom | T1053 |
| 20 | Abnormal DNS spike (tunneling) | DNS logs / Sysmon 22 | Tunneling or exfil? Query length and volume | T1071.004 |

In the next article i will share a table with SQL queriess associate with each Alert type.
## Conclusion 


As a junior, creating your own playbook is a way to practice the tons of information we learn, and to structure your thinking. This pattern helps me work through the life cycle of an alert:
 
**Read the alert → check the right logs → decide true/false positive → scope → escalate or close**
 
Next step for me: apply this on my own SIEM. I am currently deploying Wazuh in my home lab to centralize logs from Linux, web applications, and a Windows endpoint. So I can replay these scenarios against real data and turn this framework from theory into practice.


## 📫 Connect

- 🎓 **TryHackMe:** https://tryhackme.com/p/LilTyki
- 💼 **LinkedIn:** https://www.linkedin.com/in/yann-danhier/














