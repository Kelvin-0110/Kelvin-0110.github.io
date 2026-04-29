---
title: "IDOR – Unauthorized Access via Predictable Identifier Manipulation | User ID Controlled by Request Parameter"
date: 2026-03-29 13:00:00 +0530
categories: [Web Security]
tags: [
  idor,
  broken-access-control,
  predictable-identifiers,
  guid,
  horizontal-privilege-escalation,
  portswigger
]
platform: PortSwigger
author: Shivansh Sharma
image:
  path: /assets/images/posts/idor.webp
  alt: IDOR via GUID Manipulation
---

## Overview

This lab demonstrates a **horizontal privilege escalation vulnerability** where user accounts are identified using **GUIDs**.

Even though the identifiers are complex and unpredictable, the application fails to enforce proper authorization checks.

---

## Objective

Retrieve the API key of the user `carlos` and submit it to solve the lab.

---

## Reconnaissance

Logged in using:

```
wiener:peter
```

After logging in:

- Navigated to the blog section
- Observed that each post shows the **author's username**
- Clicking on the username redirects to their profile page

This indicates that user profiles are accessible via a **user ID in the URL**.

---

## Exploitation

While browsing blog posts:

- Found a post authored by `carlos`
- Clicked on the username `carlos`
- The URL revealed a **GUID associated with his account**

Example:

```
bca676dc-167a-4058-bb6b-6e66994eb5e1
```

This confirms:

- User IDs are exposed in the URL
- GUIDs are used instead of sequential IDs

---

## Proof of Exploitation

- Navigated back to my own account page
- Observed the URL structure:

```
/my-account?id=<your-user-id>
```

- Replaced my user ID with Carlos's GUID:

```
/my-account?id=bca676dc-167a-4058-bb6b-6e66994eb5e1
```

- Pressed Enter

➡️ The page loaded Carlos's account details, including his API key.

---

## Impact

- Unauthorized access to other users' accounts
- Exposure of sensitive information such as API keys
- Failure of horizontal access control

> Using GUIDs does not prevent attacks if authorization checks are missing.

---

## Mitigation

- Enforce strict server-side authorization checks
- Validate that the logged-in user owns the requested resource
- Do not rely on complex identifiers (GUIDs) for security
- Implement proper access control on all user-specific endpoints

---

## Real-World Insight

This is a textbook IDOR vulnerability.

Many developers assume that replacing numeric IDs with GUIDs is enough to prevent attacks. However, once a GUID is exposed anywhere in the application, it can be reused by an attacker.

Security should always rely on **authorization checks**, not on hiding identifiers.