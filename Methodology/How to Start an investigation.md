# How to determine a true or false positive ? A soc  analyst's starting point

## Introduction 

When i started learning SOC analyst role; i had no idea how to start a investigation.When we learn cybersecurity we don't have a company playbook, we are alone inf ront of the desk.  I learn tools, theory, improve my skill but when i started a tryhackmeroom i was like "Ok i have no idea how to start". After a few research i made a little playbook on my desk to help me memorize everything i needed. 

A Soc Analyst has to decided whether an alert is a false positive or true positive, but the challenge is to stay fast and rigorous. Answering what ? where? when? who ? why? quickly enough to keepuo without rushing so much that a real threat slips through. That balance is exactly why haven method matters/


## The 5 w and the triage 

Before trying to go through windows logs, or network logs or whatever, we context about the alert. As i said more earlier answer the 5 w give us the contact :

- What ? Which rule fire, and what exactly does it detect ?
- where ? which hosts and Ips are involves ? 
- when ? what isd the timeline ?
- Who ? which user or account is involved ?
- Why ? is this a true or false ? or benign activity ?

This context and information should give us enought information to decided if the alert is true or false , and if he needed to be escalate or close. 

Also method is what protect a soc analyst from tiredness, a soc analyst can see thousand alert per day. And after a feww hundred he become tired and less effective. That why build our own little playbook early in the career help. 


## The LOLbins : 

Sometime its easy and very fast to determine if the alert is a true positive or not. Like if a powershell is launch by an excel or contains -enc or IEX in the queries that most of the time a True positive. Understand LOLBins for windows and linux will increase the soc efficiente. Lot of time an attacker will use LOLbins (living off the land Binaries) to avoid anti-virus. A corect set of rule and the behavior user is the keys to alert these kind of exploit. 


## Into the logs : 

A single alert rarely tells a story, the real work is correlation. To understand what is behind the alert we need to correlate difference source. Take a exemple : Its 9AM during work schedule. The server get 5 SSH logins failed. From a unknow IP. These can be a remote employee who forget to connect through the VPN or an attacker who try to brute force the SSH logins. To determine which case is it we need to correlate differente informations and source.

- Check the Ip reputation and origin
- Chech the user's history : Does an employee useually work at this time ? and any employee is in remote ?
- Analyze the attempt pattern : 5 failed attempts followed by a sucessful login ? 5 failed attempts on 50 different account ?

Everything is answer whith more information we pickup in logs. As i said earlier a few months ago  the first time i investigate an alert, i was like "Ok, i know i need to check that network logs , that syslog but how ? what the query ? what's the start ? " That why i made a table. To enhance the process and answer faster. 

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

As a junior , created our own "playbook " give the opportunity to pratctie in a certains way the tons of informations we learn. And structured our think. These patterns help me to answer the alert life time :

Read the alert → check the right logs → decide true/false positive → scope → escalate or close

Next step for me: apply this on my own SIEM. I am currently deploying Wazuh in my home lab to centralize logs from Linux, web applications and a Windows endpoint — so I can replay these scenarios against real data and turn this framework from theory into practice.



## 📫 Connect

- 🎓 **TryHackMe:** https://tryhackme.com/p/LilTyki
- 💼 **LinkedIn:** https://www.linkedin.com/in/yann-danhier/














