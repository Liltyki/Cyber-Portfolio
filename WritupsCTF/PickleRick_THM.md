# Pickle Rick — TryHackMe

**Platform :** TryHackMe

**Category :** Web Exploitation / Linux

**Difficulty :** Easy

---

## Recon
First, let's start with Reconnaissance. At this stage, 
I need to gather as much information as possible. 
In this case, Nmap and View Page Source are enough !
### Nmap
```bash
nmap -sCV -Pn -O <IP>
```
-sCV = Script scan / Version Detection 

-Pn = no ping 

-O = Os detection

Classic mode for a pentesting 

<img width="880" height="661" alt="Capture d&#39;écran 2026-04-07 145026" src="https://github.com/user-attachments/assets/9c4806a2-221e-4fff-aa81-7ca99ec23846" />


**Open ports :**
- 22 — SSH
- 80 — HTTP Apache 2.4.41

### Page Source
<img width="1345" height="713" alt="Capture d&#39;écran 2026-04-07 144659" src="https://github.com/user-attachments/assets/f9dae7e9-298b-461c-90d3-044d8739dc8b" />

We found an interesting thing on the main page — the developer 
left credentials in an HTML comment ! This is a serious 
security breach. Now I have the username, I just need 
to find the password.
- Username : `R1ckRul3s`




### Domain Enumeration 

Now I need to find hidden directories and potentially 
the login page ! The perfect tool for that is Gobuster — 
it brute-forces hidden files and directories on a web 
server using a wordlist.

<img width="1081" height="236" alt="Capture d&#39;écran 2026-04-07 145338" src="https://github.com/user-attachments/assets/1d59670a-7821-4487-ad8a-cc4fcafa4a70" /><img width="726" height="239" alt="Capture d&#39;écran 2026-04-07 145431" src="https://github.com/user-attachments/assets/8446224f-6f5a-43fd-ba44-c2621793eb56" />

we find Robots.txt and login.php. Lets discover that page


### robots.txt

robots.txt is very useful in pentesting — developers often list 
hidden paths they don't want indexed, and sometimes even 
sensitive information. Unfortunately, the file is public 
and readable by anyone.

> 💡 Rule : Always check /robots.txt !

<img width="1920" height="1032" alt="Capture d&#39;écran 2026-04-07 145454" src="https://github.com/user-attachments/assets/07e860d0-176c-4ea5-88ea-a6c0c6e2c1c9" />


Found  : `Wubbalubbadubdub`

---

## Exploitation

### Login
![Screenshot_20200621_010045](https://github.com/user-attachments/assets/f92359e1-deb9-4e3f-a80c-941684ef6e8a)

Let's log in with the credentials found during Recon, using 
the word found in robots.txt as the password. 
We now have access to the Command Panel (RCE) !

In this CTF, we need to find 3 ingredients. 
In this phase we will use several commands :

```bash
whois
sudo -l
ls
```
<img width="1920" height="1032" alt="Capture d&#39;écran 2026-04-07 145608" src="https://github.com/user-attachments/assets/290c7cc2-13ac-49b4-ab71-5aa639fecf67" />

<img width="1920" height="1032" alt="Capture d&#39;écran 2026-04-07 145817" src="https://github.com/user-attachments/assets/1045c66b-4675-4c2d-8f2b-9b76da54976c" />

With these commands we know who we are, what privileges we have, 
and we can navigate the file system to find our ingredients !
The most interesting finding is from sudo -l — we can run 
sudo as www-data without any password (NOPASSWD: ALL).
This means no privilege escalation needed !


---

### Ingredient 1 :

I found Sup3rS3cretPickl3Ingred.txt — the first instinct is 
to cat it. But the command is disabled ! Let's try alternative 
commands like less, more, strings or head. 
They can't disable everything !

<img width="1920" height="1032" alt="Capture d&#39;écran 2026-04-07 145619" src="https://github.com/user-attachments/assets/3e0f0cff-d710-483a-a055-0286b5939495" />



```bash
less Sup3rS3cretPickl3Ingred.txt
```

<img width="1920" height="1032" alt="Capture d&#39;écran 2026-04-07 145643" src="https://github.com/user-attachments/assets/447fb5a8-6e09-4be7-ac8c-41fd7a81e79b" />

**Flag : mr. meeseek hair**

---




### Ingredient 2

We found a clue — the other ingredients are located in 
another user's directory. Let's navigate the file system 
and read every .txt file to find our ingredients !

```bash
less /home/rick/"second ingredients"
```



**Flag : 1 jerry tear**
<img width="1920" height="1032" alt="Capture d&#39;écran 2026-04-07 145928" src="https://github.com/user-attachments/assets/6dcbc8da-578d-4b8a-af83-887ca1a401fe" />
---



### Ingredient 3

As we discovered earlier with sudo -l, we can run any command 
as root with www-data — no privilege escalation needed !

```bash
sudo less /root/3rd.txt
```
<img width="1920" height="1032" alt="Capture d&#39;écran 2026-04-07 150032" src="https://github.com/user-attachments/assets/ea0ec2c7-4d8a-47b1-bd67-ce15f7936158" />


**Flag : fleeb juice**


----
---

## Summary

This room was a great introduction to web enumeration and basic Linux exploitation. I really enjoy doing that room first because I'm a Rick & Morty fan, and the disabled cat command 
was a fun challenge that pushed me to find alternatives !


**What I learned :**
- Always check the page source — developers sometimes leave credentials in HTML comments
- robots.txt can contain sensitive information 
- cat blocked ? Try less, more, strings, head or tac
- Always check sudo rights after gaining access : sudo -l
- www-data with NOPASSWD ALL = no privilege escalation needed !

**Attack chain :**
Page Source → robots.txt → Login → RCE → sudo -l → 3 Flags









