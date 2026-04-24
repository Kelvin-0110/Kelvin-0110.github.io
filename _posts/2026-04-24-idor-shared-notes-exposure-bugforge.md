---
title: "IDOR – Unauthorized Access to Shared Notes via Base64 ID Manipulation | BugForge"
date: 2026-04-24 1:00:00 +0530
categories: [Web Security, Broken Access Control]
tags: [idor, access-control, base64, bugforge, web-security]
platform: BugForge
author: Shivansh Sharma
image:
  path: /assets/images/posts/idor-gift-lab-bugforge.webp
  alt: IDOR BugForge Lab
---

## Overview

This lab demonstrates an **Insecure Direct Object Reference (IDOR)** vulnerability in a note-sharing feature of a web application. Users can generate shareable links for their notes, but the application fails to properly enforce access control on these shared resources.

The flaw arises from the use of **predictable, Base64-encoded identifiers**, which can be manipulated to access other users' notes.

---

## Objective

- Analyze the note-sharing functionality  
- Identify how shared links are generated  
- Manipulate identifiers to access unauthorized data  

---

## Reconnaissance

The application allows users to create notes and share them via a generated link.

Example shared URL:

```bash
https://lab-1777006964043-5899x2.labs-app.bugforge.io/share/bGlzdFdpdGhJZC0=Mg==
```

The last part of the URL appears to be an encoded identifier:

```
bGlzdFdpdGhJZC0=Mg==
```

---

## Exploitation

### Step 1: Decode the identifier

The encoded string was identified as Base64 and decoded:

```
listWithId-2
```

This reveals a pattern:

- `listWithId-<number>`

### Step 2: Modify the ID

The numeric ID was changed from `2` to `1`:

```
listWithId-1
```

### Step 3: Re-encode the modified value

The modified string was encoded back to Base64:

```
bGlzdFdpdGhJZC0x
```

### Step 4: Access unauthorized data

The modified value was inserted into the URL:

```
https://lab-1777006964043-5899x2.labs-app.bugforge.io/share/bGlzdFdpdGhJZC0x
```

**Result:**

- Successfully accessed another user's notes without authorization

---

## Proof of Exploitation

- Original user can access only their own shared notes
- By modifying the encoded ID, notes belonging to other users are exposed
- No authentication or authorization checks are enforced on the resource

---

## Impact

- Unauthorized access to sensitive user data
- Privacy breach across multiple users
- Potential exposure of confidential information stored in notes

> This type of vulnerability can scale easily if IDs are sequential and predictable.

---

## Mitigation

- Enforce proper server-side authorization checks for every request
- Avoid using predictable identifiers for sensitive resources
- Use random, non-guessable IDs (UUIDs)
- Do not rely on encoding (like Base64) for security
- Validate ownership before returning data

---

## Real-World Insight

This is a classic example of developers relying on obfuscation instead of security.

Encoding data in Base64 does not protect it. Attackers routinely decode and manipulate such values to test access control weaknesses.

Similar vulnerabilities are commonly found in:

- File sharing systems
- Document collaboration platforms
- API endpoints exposing object IDs

Understanding and exploiting IDOR is a core skill in real-world penetration testing and bug bounty hunting.