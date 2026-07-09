---
layout: post
mermaid: true
title: "Username Enumeration via Response Timing"
date: 2026-07-07
categories: [websec]
tags: [portswigger, authentication, username-enumeration, timing-attack, burp-suite]
---

## Overview

| | |
|---|---|
| **Platform** | PortSwigger Web Security Academy |
| **Difficulty** | Practitioner |
| **URL** | <a href="https://portswigger.net/web-security/authentication/password-based/lab-username-enumeration-via-response-timing" target="_blank">Username enumeration via response timing</a> |
| **Focus** | A login form that never changes its error message still leaks valid usernames — through how long it takes to respond |

This one's a good reminder that "the response looks identical" isn't the same as "the response is identical." The login form always says the same thing when you get it wrong. The clock doesn't lie the same way the error message does.

---

## The Idea: Timing as a Side Channel

The app checks credentials in two steps: first it looks up the username, then — only if that username exists — it checks the password. That second step is the expensive one, and it gets slower the longer the submitted password is. So if I submit a real username with a very long password, the app does real work and takes a while. If I submit a fake username, it never gets that far and comes back fast, no matter how long the password is.

Same error message either way. Different amount of time to produce it. That gap is the whole attack.

There's also IP-based rate limiting on failed logins, which needed dealing with before I could test usernames at any scale.

> One thing worth flagging up front: the valid username and password I found here (`agenda` / `superman`) are randomized per lab instance by PortSwigger. If you run this lab yourself, you'll almost certainly land on a different pair — it's the technique that carries over, not these two words.

---

## Establishing the Baseline

First, a normal login to see the request shape:

```
POST /login HTTP/1.1
Host: <LAB_ID>.web-security-academy.net
Cookie: session=wKT8bdev9iX9fWYQLQNu[...]
Content-Type: application/x-www-form-urlencoded
Content-Length: 30

username=wiener&password=peter
```

Straightforward — `username` and `password`, nothing unusual.

## Getting Rate-Limited (and Around It)

A handful of failed attempts in a row got me this:

```
You have made too many incorrect login attempts. Please try again in 30 minute(s)
```

Classic IP-based lockout. Poking at it, the app was trusting the `X-Forwarded-For` header to decide which IP to block — a pretty common mistake when an app assumes it's always sitting behind a proxy, without checking that the request actually came through one.

`X-Forwarded-For` exists so a proxy can tell the app "this request really came from this IP," since otherwise the app would only ever see the proxy's own address. The problem is it's just a regular header — nothing stops a client from setting it themselves. If the app doesn't verify the request came through a proxy it trusts, anyone can claim to be any IP they like.

```
X-Forwarded-For: 1.2.3.4
```

Sending that on a blocked request lifted the lockout immediately. Nudging the value slightly on every request (last octet up by one, say) meant no single IP ever built up enough failed attempts to get blocked again.

## Confirming the Timing Gap

With the lockout dealt with, I compared a short password against a very long one (~480 characters), for both a username that exists and one that doesn't:

| Username | Password length | Response time |
|---|---|---|
| `carlos123456` (fake) | short | 392 ms |
| `carlos123456` (fake) | ~480 chars | 619 ms |
| `wiener` (real) | short | 486 ms |
| `wiener` (real) | ~480 chars | **2765 ms** |

For the fake username, the long password barely registers. For the real one, it more than quintuples the response time. That's a big enough gap to automate against.

## Automating the Search with Intruder

I set up a Burp Intruder attack:

- **Attack type:** Pitchfork — walks two lists in lockstep, one item from each per request, instead of testing every combination like Cluster Bomb would
- **List 1:** candidate usernames
- **List 2:** a slightly different `X-Forwarded-For` value on every request, to dodge the lockout
- **Password (fixed for every request):** the same ~480-character junk string

Community Edition sends these one at a time instead of in parallel like Pro does, which sounds like a downside but actually helps here — concurrent requests would add network noise that muddies the timing. It just takes longer to finish.

Sorted by response time, one result stood out immediately:

```
username=agenda&password=[...480 chars]  →  2441 ms
```

Everything else sat in the normal 300–700 ms range. Ran it twice more by hand in Repeater to rule out a fluke (2441 ms, then 2771 ms) — `agenda` is a real account.

## Brute-Forcing the Password

Same Pitchfork setup, but now `username` is fixed to `agenda` and the second list is candidate passwords instead of spoofed IPs. Timing doesn't matter for this part — the tell is the **status code**. Wrong password: `200`, same login page. Right one: `302`, a redirect.

```
HTTP/2 302 Found
Location: /my-account?id=agenda
Set-Cookie: session=4cn0Wh4OwhZHL7FbRtwo[...]; Secure; HttpOnly; SameSite=None
```

Working login: `agenda:superman`.

## Logging In

One last wrinkle: my actual browser's IP was still locked out from earlier attempts, only the spoofed IPs weren't. Burp's "Request in browser (in original session)" feature on the successful Repeater request let me carry that session over into the browser directly, instead of trying to log in normally and hitting the lockout again.

That landed on `/my-account?id=agenda` — lab solved.

---

## Attack Flow

<div class="mermaid">
sequenceDiagram
    participant A as Attacker (Burp Suite)
    participant S as Login Endpoint
    A->>S: POST /login (invalid creds, repeated)
    S-->>A: Lockout - try again in 30 min
    A->>S: POST /login + X-Forwarded-For spoofed IP
    S-->>A: 200 OK - lockout bypassed
    Note over A,S: Timing calibration
    A->>S: username=invalid + long password
    S-->>A: ~600ms fast rejection
    A->>S: username=wiener valid + long password
    S-->>A: ~2700ms password check executed
    Note over A,S: Intruder - username enumeration Pitchfork
    A->>S: candidate usernames + long password + rotating IP
    S-->>A: agenda - ~2400-2700ms outlier
    Note over A,S: Intruder - password brute-force Pitchfork
    A->>S: agenda + candidate passwords + rotating IP
    S-->>A: 302 Found superman - login success
    A->>S: GET /my-account?id=agenda authenticated
    S-->>A: Account page - lab solved
</div>

---

## Key Takeaways

**Response time can leak what the response body never does.** An expensive check that only runs "if this username is real" gives away account existence through delay alone, even with an identical error message and status code.

**A header is only trustworthy if something verifies it.** Relying on `X-Forwarded-For` to know the "real" client IP, without confirming the request came through an actual trusted proxy, lets anyone claim to be any IP they want.

**Pitchfork beats Cluster Bomb when two lists should move together** — a username list paired one-to-one with a list of throwaway IPs, instead of testing every username against every IP.
