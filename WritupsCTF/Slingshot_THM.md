# 🔍 TryHackMe — Slingshot | SOC Investigation Writeup

> **Platform:** TryHackMe
> **Room:** Slingshot
> **Focus:** Elastic Stack log analysis

---

## 🎯 Scenario

Slingway Inc., a leading toy company, has recently detected suspicious activity on its e-commerce web server and potential unauthorized modifications to its database. I've been brought in to analyze the available logs and determine the scope and impact of the attack, using an Elastic Stack instance containing logs from the suspected compromise. Slingway's IT team noted that the suspicious activity began on **July 26, 2023**.

---

## 🔎 Initial Overview

Unlike the Boogeyman or Benign rooms, this one is more guided — I follow and answer each question rather than free-hunting. And this time it's not Splunk: it's **Elastic**. As stated in the scenario, the activity starts on July 26, 2023, so reviewing the application logs and the database logs is a good starting point.

## 🕵️ Who Is the Attacker?

The first questions ask me to identify the suspicious activity, so let's find the attacker's IP. The answer is pretty easy because they is only 1 IP.

<img width="750" alt="Attacker IP identified in Elastic" src="https://github.com/user-attachments/assets/3ed3302b-4375-44a4-abb8-dc9a8611a157" />

### 🔍 Reconnaissance & Enumeration

Most attacks start with reconnaissance. Passive recon is very hard to spot because the attacker doesn't interact with us  but **active recon** leaves traces. And here, the User-Agent gives 3 informations. As the screenshot shows, a **Nmap Scripting Engine** string to scan the network  **Gobuster** for directory enumeration and **Hydra** for brute-force . 3 tools for Reconnaissance enumeration and bruce forcing.

<img width="750" alt="Nmap, Gobuster and Hydra User-Agent strings in the logs" src="https://github.com/user-attachments/assets/0f7207b7-b9ba-46de-8726-bafda1962b8a" />

 Let's continue and find out what the attacker obtained, starting with the HTTP 200 responses.

<img width="750" alt="HTTP 200 responses" src="https://github.com/user-attachments/assets/667acf3a-71e1-456f-a4fd-ee409c5e5c4a" />

We get the flag,  every enumerated URL returned a 200. The next question is to find the login page  but I don't see it among the HTTP 200 responses. I remember from some Red Team rooms that **login pages often return a 401** rather than a 200, because the request requires authentication.

<img width="750" alt="Login page returning HTTP 401" src="https://github.com/user-attachments/assets/653d0c97-a3de-4868-9dc1-9534094633b9" />

### 🚪 Initial Access

With the Hydra User-Agent and the login page identified, the attacker will try to gain initial access. By inspecting the `apache_logs`, I can find the compromised account. There are a few ways to get there, I'll simply filter on `response.status:200` for the Hydra User-Agent and read the request's `Authorization` header.

<img width="750" alt="Successful login — Basic Auth header captured" src="https://github.com/user-attachments/assets/8dfc8302-a178-48f4-825b-23fb63ba83dd" />

CyberChef decodes that Base64 (Basic Auth) string to: **`admin:thx1138`**.

### 💥 Exploitation — Web Shell Upload

The attacker now has access  but what does he do once inside? The scenario points us toward a web shell upload, with the hint `/admin/upload.php`. Filtering on `http.method: POST` containing `/admin/upload.php`, we find the web shell.

<img width="750" alt="POST request uploading the web shell" src="https://github.com/user-attachments/assets/be2703c6-df40-455d-a935-1910f75ef131" />

The file is **`easy-simple-php-webshell.php`**  a primordial information, because a web shell interacts directly through the URL. We can suppose the path is something like `/uploads/easy-simple-php-webshell.php?cmd=`. Let's verify it.

<img width="750" alt="Commands executed through the web shell" src="https://github.com/user-attachments/assets/8a199ccf-0a11-40b1-be94-abbc95bdc62f" />

Only 5 commands were executed. The first is `whoami`. The last one is the most interesting: the attacker tries to reach the database via an **LFI (Local File Inclusion)** attack.

##🗄️ Database Access & Data Manipulation

phpMyAdmin is a free PHP tool for administering MySQL. For the next two questions, I continue with a simple filter containing `/phpmyadmin` to find the database name.

<img width="750" alt="phpMyAdmin access and database name" src="https://github.com/user-attachments/assets/3d47fb89-a6e9-4c05-8ace-c7e8ca2dd31b" />

The final question — *"What flag does the attacker insert into the database using import.php?"* : is the attacker's actual goal ! what does he want to inject? Filtering on `import.php` and bingo! He inserts a row into the **`credit_cards`** table.

<img width="640" height="734" alt="Capture d&#39;écran 2026-07-17 193815" src="https://github.com/user-attachments/assets/d7799a7e-6ab5-45b0-b265-0e055251ace0" />

---

## 🔗 Kill Chain

```
1. RECONNAISSANCE
   └─ Active scanning with Nmap ( "Nmap Scripting Engine" User-Agent)
   └─ Directory enumeration with Gobuster
        │  T1595 — Active Scanning

2. INITIAL ACCESS
   └─ Brute-force attack on the login page with Hydra
   └─ Compromised account recovered : admin:thx1138
        │  T1110 — Brute Force  |  T1078 — Valid Accounts

3. EXPLOITATION — Web Shell Upload
   └─ POST to /admin/upload.php
   └─ Malicious file uploaded: easy-simple-php-webshell.php
   └─ RCE via ?cmd= (system($_GET['cmd'])): 5 commands executed, starting with whoami
        │  T1505.003 — Web Shell  |  T1059 — Command Execution

4. LOCAL FILE INCLUSION (credential theft)
   └─ LFI via php://filter + path traversal on /admin/settings.php?page=
   └─ Target: /etc/phpmyadmin/config-db.php (source code exfiltrated in Base64)
   └─ MySQL database credentials stolen
        │  T1190 — Exploit Public-Facing App  |  T1552.001 — Credentials in Files

5. DATABASE ACCESS (data breach)
   └─ Authenticated access to /phpmyadmin with the stolen credentials
   └─ Sensitive database read: customer_credit_cards → table credit_cards
        │  T1213 — Data from Information Repositories  |  T1005 — Data from Local System

6. DATA MANIPULATION
   └─ INSERT into credit_cards via import.php
   └─ Flag injected in the cardholder_name field: c6aa3215a7d519eeb40a660f3b76e64c
        │  T1565.001 — Stored Data Manipulation
```

## 🎯 MITRE ATT&CK Mapping

| Tactic | Technique | ID |
|---|---|---|
| Reconnaissance | Active Scanning | T1595 |
| Credential Access | Brute Force | T1110 |
| Initial Access | Valid Accounts | T1078 |
| Initial Access | Exploit Public-Facing Application | T1190 |
| Persistence | Server Software Component: Web Shell | T1505.003 |
| Execution | Command and Scripting Interpreter | T1059 |
| Credential Access | Credentials in Files | T1552.001 |
| Collection | Data from Information Repositories | T1213 |
| Impact | Stored Data Manipulation | T1565.001 |

---

## 💭 What I Learned

This room was fairly easy but genuinely interesting. Most of the time I use Splunk for my investigations, but Elastic is an excellent open-source SIEM  and in the Blue Team world it is mandatory to be able to work across multiple SIEMs . The kill chain here is clean to follow, with a clear end goal: inserting a malicious row into the database.

## 📫 Connect

- 🎓 **TryHackMe:** https://tryhackme.com/p/LilTyki
- 💼 **LinkedIn:** https://www.linkedin.com/in/yann-danhier/








