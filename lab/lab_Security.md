# How I Secured My Cybersecurity Lab

## Introduction

As part of my cybersecurity journey, I decided to build a personal lab to practice both Red Team and Blue Team techniques. This lab runs on a VPS hosted  and uses Docker to simulate real-world attack and defense scenarios.

But building a secure lab is very important . A poorly secured lab exposed to the internet is a real target, and through this process path to secure my own lab i learned a  lot , no longer theory its time to practice. 
---

## Part 1 — The Foundation : Choosing a VPS

I chose OVH as my VPS provider for several reasons , it is a French provider, reliable, affordable, and includes built-in DDoS protection.  As OS version i choose Ubuntu 22.04 LTS, the reason to not take the last version is for the documentation the 22.04  brought and the stability to this OS. Obviously i will need to upgrade to a newest version in a few years but i'm fine for the 5 next years due to the support. 

The first thing I did after connecting for the first time was update the system :

```bash
sudo apt update && sudo apt upgrade -y
```

Never skip this step. Running outdated packages means running known vulnerabilities. With a updated Packahes the only danger for me now is Zero-Day vulnerability. 

---

## Part 2 — SSH Hardening

SSH is the main entry point to my server. Leaving it with default settings would be danger and easy to gain acces for a malicious attacker. I have in mind to change port , use a MFA (multi-factor authentification), create Key pair and change several options .

### My keys pair with ed25519  ?

I generated my SSH key pair using the **ed25519** algorithm instead of the traditional RSA. I decided to use ed25519 because in theory this is more strong and we can considere RSA as a old cryptograpy. Still usable but ed25519 make me a better choice in this case! 

```bash
ssh-keygen -t ed25519 -C "lab-ovh"
```
his command generates two files :
- id_ed25519   → the private key (never shared)
- id_ed25519.pub → the public key (copied to the VPS)

The private key stays on my machine. The public key is copied to the VPS using :
```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub ubuntu@IP -p 2222
```
This adds my public key to  ~/.ssh/authorized_keys on the server.

When I connect, SSH verifies that my private key matches the public key on the server without ever sending the private key over the network.

I also created a backup of my private key on an encrypted USB drive. If I lose my machine, I can still access my server.

So now 
### MFA on SSH — Two Factors to Connect

I configured SSH to require **both** a private key **and** a password to connect. This is Multi-Factor Authentication applied to SSH :

- **Something you have** → the private key file stored on my machine
- **Something you know** → the password

In `/etc/ssh/sshd_config` :

```
PubkeyAuthentication yes
PasswordAuthentication yes
AuthenticationMethods publickey,password
```

This means that even if an attacker steals my password, they cannot connect without the private key  and vice versa.

### Additional SSH Hardening

```
PermitRootLogin no       # Root cannot login via SSH
MaxAuthTries 5           # Max 5 attempts before disconnect
Port 2222                # Custom port — avoids automated scans on port 22
```

Changing the default port from 22 to 2222 does not add security by itself, but it significantly reduces noise from automated scanners that target port 22.

---

## Part 3 — Firewall with UFW

UFW (Uncomplicated Firewall) is my first line of defense against unwanted connections.

### Default Deny Policy

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

This is the **whitelist approach** — everything is blocked by default, and I only open what I explicitly need. This is much safer than a blacklist approach where you try to block known threats.

### Allowed Ports

```bash
sudo ufw allow 2222/tcp   # SSH
sudo ufw allow 51820/udp  # WireGuard VPN
```

That is it. Only two ports are publicly accessible. Everything else — DVWA, Wireshark, target SSH — is only accessible through the WireGuard tunnel.

---

## Part 4 — Fail2ban : Automated Brute Force Protection

Even with MFA on SSH, automated bots will still attempt to connect. Fail2ban monitors authentication logs and automatically bans IPs that fail too many times.

My configuration in `/etc/fail2ban/jail.local` :

```ini
[DEFAULT]
bantime = 3600    # Ban for 1 hour
findtime = 600    # Detection window : 10 minutes
maxretry = 5      # Max 5 failed attempts

[sshd]
enabled = true
port = 2222
logpath = /var/log/auth.log
```

**Why jail.local and not jail.conf ?**
`jail.conf` is overwritten during Fail2ban updates. Always use `jail.local` for custom configuration — it is never touched by updates.

---

## Part 5 — WireGuard VPN : My Personal Encrypted Tunnel

WireGuard transforms my VPS into a personal VPN server. It creates an encrypted tunnel between my Kali machine and the VPS.

### Why WireGuard ?

WireGuard is a modern VPN protocol integrated directly into the Linux kernel since version 5.6. It uses **ChaCha20** for encryption and **Curve25519** for key exchange. With only ~4,000 lines of code versus 600,000 for OpenVPN, its attack surface is minimal.

### How It Works

```
Kali VM (10.8.0.2)
      ↓
  [ChaCha20 encrypted tunnel]
      ↓
VPS OVH (10.8.0.1 / 135.125.191.20)
      ↓
   Internet
```

Once connected to the VPN, my real IP is hidden behind the VPS IP. My ISP only sees an encrypted connection to OVH — not what I am doing.

### The MFA Analogy

Just like SSH MFA, WireGuard uses asymmetric cryptography :

- **Private key** → stays on my machine, never shared
- **Public key** → shared with the server to verify identity

---

## Part 6 — Docker Lab Isolation

My lab runs inside Docker containers on the VPS. Each container represents a machine in a simulated network.

### Internal Network

All containers communicate on an isolated internal network `192.168.100.0/24` :

```
kali_attacker  → 192.168.100.10
target_ssh     → 192.168.100.20
dvwa_web       → 192.168.100.30
wireshark      → 192.168.100.40
```

No traffic leaves this network to the internet — it is completely isolated.

### Binding Ports to WireGuard Only

By default, Docker exposes ports on all interfaces — meaning anyone on the internet could reach my vulnerable containers. I fixed this by binding ports exclusively to the WireGuard interface :

```yaml
ports:
  - "10.8.0.1:8080:80"    # DVWA — WireGuard only
  - "10.8.0.1:2223:22"    # Target SSH — WireGuard only
  - "10.8.0.1:3000:3000"  # Wireshark — WireGuard only
```

Now only someone connected to my WireGuard VPN can access these services.

### Continuous Traffic Capture

I configured a TCPDump container that captures all network traffic continuously :

```yaml
tcpdump:
  image: nicolaka/netshoot
  network_mode: "host"
  command: tcpdump -i any -w /captures/capture_$(date).pcap
```

Combined with a daily cron job that deletes captures older than 24 hours, this gives me a rolling log of all lab activity without filling up the disk.

---

## Summary

| Security Layer | Protection |
|---|---|
| SSH ed25519 + MFA | Strong authentication |
| Port 2222 | Reduces automated scan noise |
| UFW default deny | Minimal attack surface |
| Fail2ban | Automated brute force protection |
| WireGuard VPN | Encrypted access + IP masking |
| Docker port binding | Lab services invisible from internet |
| TCPDump continuous | Full traffic visibility |

---

## What I Learned

Security is not a single action — it is a stack of layers. Each layer alone can be bypassed, but combined they make the attacker's job significantly harder.

The weakest link in my setup remains my Kali machine — if it is compromised, the attacker gets my private SSH key and WireGuard key. This is why endpoint security matters as much as server security.

> *"A chain is only as strong as its weakest link."*

---

*This lab was built as part of my cybersecurity studies. All techniques described are used in an isolated, controlled environment for educational purposes.*
