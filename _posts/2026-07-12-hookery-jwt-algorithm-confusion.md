---
title: "JWT Algorithm Confusion Leads to Platform Admin Access | Hookery"
date: 2026-07-12 13:31:00 +0530
categories: [A07 - Authentication Failures, JWT]
tags: [jwt, algorithm-confusion, rs256, hs256, public-key, privilege-escalation, webverse-pro]
platform: WebVerse Pro
author: Shivansh Sharma
image:
  path: /assets/images/posts/hookery-jwt-algorithm-confusion.webp
  alt: "Hookery JWT algorithm confusion leading to platform administrator access"
---

## Lab Link

[Hookery](https://dashboard.webverselabs-pro.com/mystery-challenges/hookery)

---

## Overview

**Hookery** was a WebVerse Pro mystery challenge built around a webhook service and a protected platform console.

The application used JSON Web Tokens for authentication. Normal user sessions were signed with RSA using the `RS256` algorithm and contained a role claim such as `customer`.

During reconnaissance, the application exposed its webhook verification public key through a publicly accessible well-known endpoint. The same key material was also accepted by the JWT verification logic.

Because the backend trusted the algorithm supplied inside the JWT header, it was possible to change the token algorithm from `RS256` to `HS256` and use the exposed RSA public key as an HMAC secret.

After changing the role claim to `admin` and signing the modified token with the public key, the application accepted the forged session and granted access to the platform administrator console.

---

## Objective

The objective was to gain unauthorized access to the protected platform console and retrieve the challenge flag.

The attack required:

1. Identifying the authentication mechanism.
2. Obtaining a valid low-privileged JWT.
3. Discovering the exposed RSA public key.
4. Exploiting JWT algorithm confusion.
5. Forging a token with an administrative role.

---

## Vulnerability Identification

The challenge briefing referenced webhook signing and dashboard authentication.

The application exposed a webhook verification key at:

```text
/.well-known/webhook-key.pem
```

A related documentation endpoint also revealed a webhook key identifier similar to:

```text
key_fe7e2f8142aa7b02
```

Registering a normal account produced an authentication cookie named:

```text
hk_session
```

The cookie contained a JWT with a header similar to:

```json
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "wh_2026a"
}
```

The payload contained ordinary user identity fields and a low-privileged role:

```json
{
  "role": "customer"
}
```

Attempting to access the platform console with the original token returned a `403 Forbidden` response indicating that the route was restricted to platform administrators.

This confirmed that authorization was controlled by claims inside the JWT.

---

## Recon and Approach

The initial reconnaissance focused on public routes, documentation, authentication behavior, and exposed cryptographic material.

The important findings were:

- A webhook RSA public key was publicly downloadable.
- User authentication relied on an `RS256` JWT.
- The token contained a modifiable `role` claim.
- The protected platform route trusted the role stored inside the JWT.
- The backend appeared to accept multiple JWT algorithms.

The likely vulnerability was an **RS256-to-HS256 algorithm confusion attack**.

In a secure implementation:

- `RS256` uses a private RSA key to sign tokens.
- The server uses the corresponding public RSA key only to verify signatures.
- `HS256` uses the same shared secret for both signing and verification.

If a vulnerable verifier accepts the JWT header's `alg` value without enforcing the expected algorithm, an attacker can switch the token to `HS256`.

The exposed RSA public key can then be treated as an HMAC secret. If the server also uses those same public-key bytes during verification, the forged signature is accepted.

---

## Exploitation

### 1. Create a Normal User Account

A standard account was created to obtain a legitimate session token.

After authentication, the application returned an `hk_session` cookie containing an `RS256` JWT.

The token was decoded locally to inspect its header and payload.

Example structure:

```text
HEADER.PAYLOAD.SIGNATURE
```

The original role was:

```json
{
  "role": "customer"
}
```

### 2. Download the Public Key

The webhook verification key was downloaded from:

```text
/.well-known/webhook-key.pem
```

The response contained an RSA public key in PEM format:

```text
-----BEGIN PUBLIC KEY-----
REDACTED
-----END PUBLIC KEY-----
```

Although public keys are normally safe to expose, they become dangerous when a JWT verifier incorrectly allows them to be used as HMAC secrets.

### 3. Modify the JWT Header

The original JWT used:

```json
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "wh_2026a"
}
```

The algorithm was changed to:

```json
{
  "alg": "HS256",
  "typ": "JWT",
  "kid": "wh_2026a"
}
```

The original key identifier and token structure were retained to minimize unnecessary changes.

### 4. Modify the Role Claim

The payload was updated from:

```json
{
  "role": "customer"
}
```

to:

```json
{
  "role": "admin"
}
```

All other harmless identity and session fields were preserved.

### 5. Sign Using the RSA Public Key

The modified JWT was signed using `HS256`, with the exposed PEM public key bytes used as the HMAC secret.

Conceptually:

```python
forged_token = jwt.encode(
    payload,
    public_key_pem,
    algorithm="HS256",
    headers={
        "kid": "wh_2026a",
        "typ": "JWT"
    }
)
```

Some JWT libraries prevent asymmetric keys from being used directly as HMAC secrets. In that case, the token can be constructed manually or generated with a lower-level HMAC implementation.

The exact public-key representation mattered. The successful version used the same public-key bytes expected by the vulnerable server-side verifier.

### 6. Replace the Session Cookie

The legitimate `hk_session` cookie was replaced with the forged token:

```http
Cookie: hk_session=<FORGED_HS256_JWT>
```

The protected platform console was then requested again.

This time, the server accepted the forged signature, trusted the attacker-controlled `role: admin` claim, and returned the administrative interface.

---

## Proof and Flag

The forged administrative JWT successfully bypassed the platform authorization check.

The platform console displayed the challenge flag.

```text
WEBVERSE{REDACTED}
```

The flag has been intentionally redacted from this writeup.

---

## Root Cause

The root cause was insecure JWT verification logic.

The application trusted the `alg` value supplied in the untrusted JWT header and did not enforce that session tokens must use `RS256`.

As a result, the verifier accepted an attacker-controlled `HS256` token and reused an RSA public key as an HMAC verification secret.

The vulnerable design effectively behaved like:

```text
Read alg from JWT header
Use configured key with whichever algorithm the attacker selected
Trust role claim after signature verification
```

This created a classic JWT algorithm confusion vulnerability.

---

## Impact

An attacker able to obtain any valid low-privileged JWT could forge arbitrary session tokens.

The impact included:

- Authentication bypass.
- Privilege escalation from customer to administrator.
- Unauthorized access to the platform console.
- Exposure of sensitive administrative data.
- Potential compromise of webhook or platform operations.
- Complete loss of trust in JWT-based authorization.

Because authorization decisions depended directly on attacker-controlled token claims, successful exploitation resulted in full administrative access.

---

## Mitigation

### Enforce a Fixed JWT Algorithm

The server must explicitly require `RS256` for session tokens.

The verification function should not select an algorithm based solely on the JWT header.

```text
Allowed algorithms: RS256 only
```

### Use Separate Keys for Separate Purposes

Webhook verification keys and authentication keys should not be reused across different security boundaries.

Separate key pairs should be maintained for:

- User authentication.
- Webhook signature verification.
- Internal service authentication.

### Reject Symmetric Algorithms for RSA Keys

The JWT library configuration should reject `HS256` whenever the application expects an RSA public key.

### Validate Claims Server-Side

Sensitive authorization claims should be strictly validated.

The server should verify:

- Issuer.
- Audience.
- Token type.
- Expiration.
- Subject.
- Role or permission scope.

### Rotate Exposed or Reused Keys

Any key used by the vulnerable verifier should be rotated after the issue is fixed.

### Keep JWT Libraries Updated

Applications should use maintained JWT libraries with safe algorithm-selection behavior and secure defaults.

---

## Lessons

- A public key is not automatically harmless when cryptographic algorithms are confused.
- JWT headers are attacker-controlled and must never determine verification policy.
- Authentication and webhook verification should use separate key material.
- Authorization claims such as `role` are only trustworthy when signature verification is implemented correctly.
- A valid low-privileged token is often enough to investigate JWT privilege-escalation flaws.
- Algorithm allowlists are essential for secure JWT verification.
