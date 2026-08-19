---
layout: post
mermaid: true
title: "Biohazard CTF: Unraveling a Resident Evil Puzzle Box"
date: 2026-08-19
categories: [ctf, puzzle, steganography, web, privesc]
tags: [puzzle-ctf, encoding, steganography, vigenere-cipher, gpg, resident-evil]
---

## Overview

| | |
|---|---|
| **Platform** | TryHackMe |
| **Difficulty** | Medium |
| **URL** | <a href="https://tryhackme.com/room/biohazard" target="_blank">https://tryhackme.com/room/biohazard</a> |
| **Focus** | Multi-layer encoding puzzles, steganography, Vigenère ciphers, and privilege escalation |

Biohazard is unlike typical boot-to-root rooms. Instead of a direct exploitation chain, this challenge presents a **puzzle-heavy environment** where each piece must be decoded, deconstructed, or extracted before you can progress. Wrapped in a Resident Evil narrative, the room requires systematic enumeration, multi-layer decoding, and careful attention to detail. This writeup walks through the complete journey from initial reconnaissance to root access.

---

## Reconnaissance

I started with a standard port scan to identify active services:

```bash
nmap -sS -sC -sV -O -Pn -p 21,22,80 <TARGET_IP>

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
80/tcp open  http    Apache httpd 2.4.29
```

Three services presented themselves: FTP, SSH, and HTTP. Both FTP and SSH would require credentials later in the chain, so the HTTP service on port 80 became my primary focus.

---

## Enumeration & Web Discovery

I began by fuzzing directories on the web server:

```bash
gobuster dir -u http://<TARGET_IP>/ -w /usr/share/wordlists/dirb/common.txt \
  -x html,php,txt,js,json,bak,old,zip -b 400,403,404

/attic/
/mansionmain/
/diningRoom/
/teaRoom/
/artRoom/
/barRoom/
/diningRoom2F/
/tigerStatusRoom/
/galleryRoom/
/studyRoom/
/armorRoom/
```

The results revealed a map of themed "rooms," each navigable and each containing fragments of the story. Exploring the first room, I found an HTML comment:

```html
<!-- SG93IGFib3V0IHRoZSAvdGVhUm9vbS8= -->
```

The format (only A-Za-z0-9+/=) immediately suggested base64 encoding. Decoding:

```bash
echo "SG93IGFib3V0IHRoZSAvdGVhUm9vbS8=" | base64 -d

How about the /teaRoom/
```

This pattern would repeat throughout the web enumeration: **puzzles are hidden in HTML comments, and each solution points to the next room.** Following these breadcrumbs, I collected flags from multiple rooms:

| Room | Flag |
|---|---|
| `/diningRoom/emblem.php` | `emblem{REDACTED}` |
| `/teaRoom/master_of_unlock.html` | `lock_pick{REDACTED}` |
| `/barRoom.../musicNote.html` (base32 encoded) | `music_sheet{REDACTED}` |
| `/barRoom.../gold_emblem.php` | `gold_emblem{REDACTED}` |
| `/diningRoom/sapphire.html` | `blue_jewel{REDACTED}` |

**A critical lesson emerged early:** When a form submission returns "Nothing happen" via `GET`, don't assume the endpoint is dead. I discovered that one endpoint required `POST` with the exact `name` attribute from the HTML `<input>` tag. Inspecting the form revealed the correct method.

---

## Combining the Crests: Multi-Layer Decoding

With flags collected, I could now unlock additional rooms. Each yielded a fragment encoded multiple times with different methods:

| Fragment | Encoding Chain |
|---|---|
| Crest 1 | base64 → base32 |
| Crest 2 | base32 → base58 |
| Crest 3 | base64 → binary → hex |
| Crest 4 | base58 → hex |

The puzzle notes indicated the correct order to concatenate these fragments. After decoding each according to its stack and concatenating:

```bash
printf '%s' "<CREST1_DECODED>" "<CREST2_DECODED>" "<CREST3_DECODED>" "<CREST4_DECODED>" | base64 -d

FTP user: hunter, FTP pass: REDACTED
```

**Key insight:** `printf '%s'` avoids newlines between fragments—using `echo` would break the final base64 decoding.

---

## Vigenère Cipher and Hidden Doors

Before accessing FTP, I needed to unlock the encrypted rooms. Submitting one of the collected flags to its corresponding form endpoint returned a Vigenère-encrypted string:

```
wpbwbxr wpkzg pltwnhro, txrks_xfqsxrd_bvv_fy_rvmexa_ajk
```

Testing Caesar rotations yielded nothing. The repeated letter patterns suggested **Vigenère cipher**—a polyalphabetic substitution. The key was revealed by the room itself (the name of the very room whose form I'd submitted the flag to). Using CyberChef with `Vigenère Decode`:

```
Decrypted: shield_key location revealed
Shield key: REDACTED
```

This key unlocked additional rooms. One in particular, `/hidden_closet/`, contained a second encrypted message and a direct hint:

```
SSH password: REDACTED
```

---

## Steganography: Three Methods, One Passphrase

With FTP credentials, I downloaded:
- Three JPG images
- One encrypted GPG file (`helmet_key.txt.gpg`)
- A narrative note mentioning the helmet key's importance

Each image contained a fragment of the GPG passphrase, but **each used a different steganographic technique**:

**Image 1 — steghide:**

```bash
stegseek --crack 001-key.jpg /usr/share/wordlists/rockyou.txt

key-001.txt → cGxhbnQ0Ml9jYW
```

**Image 2 — EXIF metadata:**

```bash
exiftool 002-key.jpg | grep Comment

Comment: 5fYmVfZGVzdHJveV9
```

**Image 3 — ZIP concatenated to JPEG:**

```bash
binwalk 003-key.jpg
unzip 003-key.jpg

key-003.txt → 3aXRoX3Zqb2x0
```

Concatenating and decoding:

```bash
printf '%s' "cGxhbnQ0Ml9jYW" "5fYmVfZGVzdHJveV9" "3aXRoX3Zqb2x0" | base64 -d

plant42_can_be_destroy_with_vjolt
```

This was the GPG passphrase.

---

## GPG Decryption and SSH Access

With the passphrase, I decrypted the GPG file:

```bash
gpgconf --kill gpg-agent
gpg --decrypt helmet_key.txt.gpg

Enter passphrase: plant42_can_be_destroy_with_vjolt

helmet_key{REDACTED}
```

(Note: The first `gpgconf` command clears the gpg-agent cache, which can otherwise mask a correct passphrase as incorrect on retry.)

The helmet key, combined with the shield key from the Vigenère solution, unlocked the final pieces: the SSH username (from a compressed archive) and the SSH password (from a text file). SSH access was now possible:

```bash
ssh umbrella_guest@<TARGET_IP>

umbrella_guest@umbrella_corp:~$ id
uid=1001(umbrella_guest) gid=1001(umbrella) groups=1001(umbrella)
```

---

## Post-Access Enumeration

Once inside, I ran `linpeas.sh` to assess the system. Two critical findings emerged:

1. **User `weasker` belongs to the `sudo` group** with unrestricted permissions.
2. **A hidden directory** in my home (`.jailcell/`) contained a narrative note that revealed the key to the second Vigenère cipher—the one from the `/hidden_closet/` web room that I couldn't decrypt until now.

With this new key, the earlier encrypted message from `/hidden_closet/` decoded to:

```
weasker login password, REDACTED
```

---

## Privilege Escalation

Lateral movement to `weasker` was straightforward:

```bash
su weasker

Password: REDACTED

weasker@umbrella_corp:~$ sudo -l

User weasker may run the following commands on umbrella_corp:
    (ALL : ALL) ALL
```

No restrictions. Root access was immediate:

```bash
sudo bash

root@umbrella_corp:~# id
uid=0(root) gid=0(root) groups=0(root)

root@umbrella_corp:~# cat /root/root.txt

[Narrative text about the team's escape]
flag: REDACTED
```

---

## Attack Chain

<div class="mermaid">
graph TD
 subgraph RECON["RECON"]
  A["Port scan: FTP SSH HTTP"]
 end
 
 subgraph ENUM["ENUMERATION"]
  B["Web directory fuzzing"] --> C["HTML comments base64 decoded"]
  C --> D["Collect flags from rooms"]
  D --> E["Vigenere cipher discovered"]
  E --> F["Combine multi-layer crests"]
  F --> G["FTP credentials obtained"]
 end
 
 subgraph EXPL["EXPLOITATION"]
  H["FTP access: download images GPG"] --> I["Steganography multi-method"]
  I --> J["Reconstruct GPG passphrase"]
  J --> K["GPG decryption helmet_key"]
  K --> L["SSH credentials unlocked"]
  L --> M["SSH access as umbrella_guest"]
 end
 
 subgraph PRIVESC["PRIVESC"]
  N["Post-access enumeration linpeas"] --> O["Second Vigenere key discovered"]
  O --> P["Lateral move to weasker"]
  P --> Q["Sudo unrestricted permissions"]
  Q --> R["Root shell obtained"]
 end
 
 A --> B
 G --> H
 M --> N
</div>

---

## Key Takeaways

1. **Puzzle-style CTFs reward systematic exploration.** Unlike exploit-based challenges, Biohazard requires thorough enumeration and careful tracking of interdependencies between clues.

2. **Multi-layer encoding is common in puzzles.** Don't assume a single decode will work—use CyberChef's "Magic" function to identify encoding chains, then apply them in sequence.

3. **Steganographic techniques are diverse.** When one method fails on multiple files, switch approaches. steghide, EXIF metadata, and format concatenation are all valid—and may all be used in the same challenge.

4. **Context matters for decryption.** Vigenère keys sourced from narrative content or discovered elsewhere on the system beat brute force every time. Always explore thoroughly before resorting to dictionary attacks.

5. **Group membership is as dangerous as sudo.** A user in the `sudo` group with no restrictions is equivalent to root access once their credentials are compromised.

6. **Details matter:** HTML comments, metadata fields, and appended data are often overlooked. Systematic inspection pays off.
