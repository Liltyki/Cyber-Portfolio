# 🔬 Personal Cybersecurity Lab


## 📖 Introduction

One of the most important steps when entering the cybersecurity field is **building your own lab**.

Having a personal lab matters for several reasons. It allows you to develop hands-on skills, host virtual machines, simulate both **Red Team** and **Blue Team** scenarios, and understand technologies you will face in real-world environments.

---

## 🎯 Goals

At the time of writing, my objectives for this lab are:

| Goal | Description |
|------|-------------|
| 🖥️ **Offload heavy tasks** | Run resource-intensive operations such as brute force attacks from a dedicated VPS |
| ⚔️ **Attack & Defense simulations** | Simulate Red Team and Blue Team scenarios using Docker containers |
| 🔒 **Personal VPN** | Enhance privacy and secure access to the lab via a WireGuard VPN |

---

## 🏗️ Architecture

```
[Kali Linux - Local VM]
        |
        | WireGuard Tunnel (encrypted)
        |
[VPS OVH - Ubuntu 22.04]
        |
   +---------+----------+
   |         |          |          
[Kali]   [Target]   [DVWA]   
.10       .20         .30       
        Docker Internal Network
        192.168.100.0/24
```

---

## 🔐 Security Configuration

- **SSH** — Port 2222, MFA enforced (public key + password)
  - *Something you have* → Private SSH key
  - *Something you know* → Password
- **Firewall** — UFW with default deny incoming policy
- **Fail2ban** — Auto-ban after 5 failed attempts (1h ban)
- **Root login** — Disabled (`PermitRootLogin no`)

---

## 🛠️ Stack

| Technology | Role |
|-----------|------|
| Ubuntu 22.04 LTS | VPS Base OS |
| WireGuard | Personal VPN |
| Docker | Lab environment isolation |
| UFW | Firewall |
| Fail2ban | Brute force protection |
| Kali Linux | Attack machine |
| DVWA | Vulnerable web application target |

