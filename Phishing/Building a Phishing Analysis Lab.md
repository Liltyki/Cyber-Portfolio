# Building a Phishing Analysis Lab

> **Part 1 of 3** — [Phishing Analysis Project](README.md)
> Next: [Analysing a phishing email](02-email-analysis.md)

---

Triaging reported emails is one of the things a SOC Tier 1 analyst does most often. Someone forwards a suspicious message, and you have to decide fairly quickly whether it's a real threat, how far it went, and what to do about it.

I've done that kind of analysis before on TryHackMe rooms, but always on samples someone else had prepared. I wanted to generate my own — control what goes into a message, then look at it from the other side and see what a defender would actually have to work with.

That meant building something. This is how I did it, and what broke along the way.


## What was already there

I didn't start from nothing. My [home lab](../Lab/README.md) already had most of what I needed: a VPS running Docker, WireGuard for remote access, UFW, fail2ban, SSH hardened onto a non-standard port, and DVWA sitting behind the VPN for web security practice.

That existing setup shaped the whole design. I already had a working private network, so the question wasn't "how do I secure this?" but "how do I fit these new services into what's already segmented?"

## What I added

Two pieces.

**Gophish** is the campaign framework. It sends the emails, hosts the landing page, and logs what happens: who opened, who clicked, who submitted credentials. It's the same tool used for corporate awareness campaigns.

**Mailpit** is a mail sink. It speaks SMTP and captures everything sent to it, but never delivers anything. You read the messages in a web interface and export them as `.eml` files.

That second choice mattered more than it looks. I could have set up a real mail server, but then messages could actually go somewhere. With a sink, they physically cannot. I'd rather the lab be safe because of how it's built than because I remembered to be careful.

```
                    VPS (Ubuntu)
    ┌──────────────────────────────────────────────┐
    │                                              │
    │   [ Gophish ]              [ Mailpit ]       │
    │    admin  10.8.0.1:3333     SMTP  127.0.0.1  │
    │    phish  10.8.0.1:8081     UI    10.8.0.1   │
    │         │                                    │
    │         └──► events ──► gophish.db           │
    │                                              │
    │   [ DVWA 10.8.0.1:8080 ]  (existing)         │
    └──────────────────────────────────────────────┘
              ▲
              │  WireGuard — the only way in
              │
         [ Analyst workstation ]
```

## Where things listen, and why

This is the part I spent the most time on, and it's the part I'd actually talk about in an interview.

When a service starts, it picks a network interface to listen on, and that choice decides who can reach it. Bind to `0.0.0.0` and it's on every interface, including the public one — anyone on the internet can knock. Bind to `127.0.0.1` and only the machine itself can. Bind to `10.8.0.1`, my WireGuard address, and only machines on the VPN can.

So nothing here listens on the public IP. The Gophish admin panel holds my password, my campaigns and my database — leaving that on the open internet means it gets found by scanners within hours. It sits on the VPN instead.

The SMTP port goes further and sits on `127.0.0.1`. Gophish runs on the same box, so it has no reason to be reachable at all, not even over the VPN.

I could have left ports open and filtered them with firewall rules instead. Binding felt safer: there's no rule to forget or get around, the service simply doesn't exist anywhere else.

The command that shows all of this:

```bash
sudo ss -tlnp
```

`-t` for TCP, `-l` for listening, `-n` for numeric ports, `-p` for the owning process. It tells you exactly what a machine exposes. It's usually the first thing I check now.

## A trap I walked into

I first tried restricting access to Mailpit with a UFW rule, then checked whether it worked. It didn't. I could still reach the service from outside.

Docker writes its own iptables rules into the `DOCKER-USER` chain, and that chain is traversed before UFW's `INPUT` rules. So when you publish a container port with `-p`, Docker opens it to the world and your firewall rules are quietly ignored. Apparently this is how a lot of databases end up exposed by accident.

The fix was to stop filtering after the fact and bind the container properly:

```bash
# instead of:  -p 8025:8025           → 0.0.0.0, exposed
# I used:      -p 10.8.0.1:8025:8025  → VPN only
```

What I took from this isn't really the Docker detail. It's that I only found out because I tested the rule instead of assuming it worked.

## Everything that broke

The setup itself took minutes. Getting it to actually work took a couple of hours, and the failures turned out to be the useful part.

**The admin panel wouldn't load.** The service was on `127.0.0.1:3333`, so it was unreachable from my workstation — which was the whole point, but I still needed a way in. There were two options: an SSH tunnel forwarding a local port through to the server, or binding it to the VPN address like everything else. I went with the VPN, since it was already running and it meant one access method for the whole lab instead of two.

**The SSH tunnel wouldn't open either.** I'd hardened SSH onto port 2222 months ago and completely forgotten. My tunnel command was quietly trying port 22 and getting nowhere. I found it by reading an `ss` output and noticing `sshd` listening somewhere unexpected.

**Gophish kept listening on the old address** after I edited the config. The process was still the one I'd started before the change — a service reads its configuration at startup, never while running. Obvious in hindsight.

**The landing page port wasn't listening at all.** I'd put both the admin server and the phishing server on 3333. They're two separate services with two separate jobs: one is the console I work in, the other is the page a target would see. Moving the landing page to 8081 also avoided a collision with DVWA, which already had 8080.

**Then it just hung.** No error, just a spinning tab. That's a different symptom from "connection refused" — refused means the packet arrived and nothing was listening, hanging means something is dropping it silently. To find out which side was broken, I tested from the server itself:

```bash
curl -k -m 5 https://10.8.0.1:3333
```

It returned a redirect to `/login`, which meant the service was perfectly healthy and the problem was between the two machines, not in the application. One command cut the search space in half.

**Finally, a protocol mismatch.** `Client sent an HTTP request to an HTTPS server`. The admin panel runs with `use_tls: true`, so it only speaks HTTPS. I'd typed `http://`.

None of this is glamorous. But interface binding, firewall behaviour, a forgotten SSH port, a config that doesn't reload, and knowing the difference between a timeout and a refused connection — that's a fair description of infrastructure security work.

## What the lab produces

Three kinds of artefact come out of a campaign, and each feeds a different part of the analysis.

The **`.eml` files** are the main one. Mailpit lets you download the raw message with complete headers, which is what article 02 works from.

The **campaign events** live in Gophish's SQLite database — sent, opened, clicked, submitted, with timestamps and recipients:

```bash
sqlite3 -header -csv /opt/gophish/gophish.db \
  "SELECT * FROM events;" > gophish_events.csv
```

Those are the same metrics a security team reports to management after an awareness campaign: click rate, credential submission rate, time to first click.

And **network traffic**, captured while the campaign runs:

```bash
sudo tcpdump -i any -w phish_lab.pcap port 8081
```

## What this lab can't show

Worth being upfront about, because it changes how article 02 is written.

**No SPF, DKIM or DMARC results.** Those checks are performed by the *receiving* mail server against DNS records — the recipient side queries the sender's domain to verify the source is authorised and the signature is valid. A mail sink does none of that, and `.test` domains have no DNS entries by design. So no `Authentication-Results` header is ever produced. There's nothing to analyse. I'll cover those mechanisms against a real-world sample instead.

**Open rates are unreliable.** Tracking pixels only fire when a real mail client fetches remote images, and a capture interface doesn't behave like Outlook. This actually mirrors production, where most clients block remote content by default and open rates are systematically undercounted — but it means the number means little here.

## Next

The lab produces samples. [Article 02](02-email-analysis.md) takes one apart: headers, sender discrepancies, URL extraction, IOCs, and where it lands on MITRE ATT&CK.

---

**Tools:** Gophish · Mailpit · Docker · WireGuard · UFW · SQLite · tcpdump

🎓 [TryHackMe](https://tryhackme.com/p/LilTyki) · 💼 [LinkedIn](https://www.linkedin.com/in/yann-danhier/)
