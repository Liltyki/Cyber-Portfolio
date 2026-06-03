# 🔍 TryHackMe — Benign | SOC Investigation Writeup

> **Platform**: TryHackMe<br>
> **Room**: Benign<br>
> **Focus**: Splunk log analysis — Windows Event ID 4688

---

## 🎯 Scenario

One of the client’s IDS indicated a potentially suspicious process execution indicating one of the hosts from the HR department was compromised. Some tools related to network information gathering / scheduled tasks were executed which confirmed the suspicion. Due to limited resources, we could only pull the process execution logs with Event ID: 4688 and ingested them into Splunk with the index win_eventlogs for further investigation. 

---

## 🏢 Network Information

The network is divided into three logical segments. This segmentation will help structure the investigation.

**IT Department**
- James
- Moin
- Katrina

**HR Department**
- Haroon
- Chris
- Diana

**Marketing Department**
- Bell
- Amelia
- Deepak

---

## 🔎 Initial Overview

Rather than answering every question like a robot, my approach was to understand what is happening in these Splunk logs.

In this scenario, I received a SOC alert for:

- A potentially suspicious process execution from the HR department
- One suspicious scheduled task creation

But one specificity: I only have access to Event ID 4688. That limits my deep search! so let's go in!

---

## My First Step in the Alert!

First I want to do 2 things: check the number of UserName and investigate the scheduled tasks!

<img width="1909" height="671" alt="UserName overview" src="https://github.com/user-attachments/assets/50571cc9-5ada-4290-b063-e004d64b6223" />

The scenario gives us the exact number of employees  (like a mini Asset Inventory) 3 from each department. But I saw 11 usernames (it's normal to have SYSTEM, so 10 is normal 11 isn't normal). **Amel1a** is very suspicious, a clear masquerading of the Amelia username from the Marketing Department. 

<img width="1629" height="562" alt="Amel1a investigation" src="https://github.com/user-attachments/assets/1aa4a218-2c75-46eb-8173-b9f8e05192b6" />

In my investigation of Amel1a I saw the attacker used a workstation from the HR department (HR_02) and only 1 event running `whoami.exe`  a reconnaissance process! The attacker used only 1 process as Amel1a, so I need to discover what the attacker did before. 

Who created Amel1a? Unfortunately the scenario doesn't give me enough information... But I have to check the scheduled tasks as the alert says!

Many different tasks, but the suspicious one is from **Chris.Fort**. Why? For many reasons: he is from the HR department, and he uses the HostName HR_02 the same as Amel1a! 

<img width="1610" height="545" alt="Chris.fort scheduled task" src="https://github.com/user-attachments/assets/b880f20e-9f79-4520-922e-abfa5c63bf60" />

Bingo, that's it !And  the CommandLine is malicious:

```
/create /tn OfficUpdater /tr "C:\Users\Chris.fort\AppData\Local\Temp\update.exe" /sc onstart
```

Why is this command malicious? 4 red flags:

- **`/tn OfficUpdater`** → misspelling
- **The location in `...\Local\Temp`** → never a legitimate path
- **`/sc onstart`** → persistence on every restart
- **`update.exe`** → generic name

In the real world, I would check every action from Chris.Fort and figure out how the attacker accessed that user account. But I can't, because I only have access to Event ID 4688.

---

### What can I find more?

The attacker needed to download the payload from the network, so I'm going to check if I can find something with that filter:

```spl
index=win_eventlogs EventID=4688 HostName=HR_02
    (ProcessName=*certutil*
     OR ProcessName=*bitsadmin*
     OR ProcessName=*curl*
     OR ProcessName=*wget*
     OR ProcessName=*powershell*
     OR ProcessName=*mshta*
     OR ProcessName=*regsvr32*
     OR ProcessName=*rundll32*
     OR ProcessName=*msiexec*
     OR CommandLine=*DownloadString*
     OR CommandLine=*DownloadFile*
     OR CommandLine=*Invoke-WebRequest*
     OR CommandLine=*urlcache*
     OR CommandLine=*BitsTransfer*)
| table _time UserName ProcessName CommandLine
| sort _time
```

<img width="862" height="485" alt="Certutil download finding" src="https://github.com/user-attachments/assets/72c8dbb0-6e2e-4226-8d06-be94df7fd148" />

The attacker used another account from the HR department, and used `certutil` to download `Benign.exe` from `https://controlc.com/e1d00035`. 

I can open this URL in a secure environment to make some search of that .exe. Make some static and Dynamic Analysis .

<img width="1560" height="383" alt="Capture d&#39;écran 2026-06-03 180537" src="https://github.com/user-attachments/assets/0a96b393-35b4-44fe-8e52-fb2f166bf8bb" />


But now the investigation ends here... the Splunk data doesn't give me the opportunity to go deeper. And i can make any analysis from that file , its only a flag for the question And now  I have all the information I need to answer the questions.



<img width="853" height="729" alt="Final result" src="https://github.com/user-attachments/assets/64aea5c6-9966-473c-ac4a-b859b808896d" />




### What i understand in the siuation §

An attacker gained access to the HR department and used lateral 
movement to reach Chris and Haroon. With Haroon, he downloaded a 
payload, and with Chris, he created a backdoor through a scheduled 
task. He also created the Amel1a account to perform reconnaissance.

That's it ! the scenario doesn't give me any other information. 

I've reached every goal!
---

### What's next?

In a real situation, we would need to:

- Clean every compromised workstation
- Find any other potential backdoors
- Check if the compromise is limited to the HR department or has spread
- Understand how the attacker initially gained access to the HR department
- Understand every action of the attacker while he was inside the network

---

### What I learned

I really enjoyed this simulation. Benign gave me the opportunity to practice like a real SOC analyst. Unfortunately, the limits of the scenario were reached very quickly every time I tried to go deep,like "who created Amel1a?" or "how did the attacker gain access to the HR department?".

But it's a good exercise to practice Splunk and improve the analyst mindset!


---

## 📫 Connect

- 🎓 **TryHackMe**: https://tryhackme.com/p/LilTyki
- 💼 **LinkedIn**: https://www.linkedin.com/in/yann-danhier/




