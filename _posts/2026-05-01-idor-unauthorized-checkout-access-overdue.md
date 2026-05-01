---
title: "IDOR – Unauthorized Access to Borrower Records | Overdue"
date: 2026-05-01 20:00:00 +0530
categories: [Web Security, Access Control]
tags: [idor, access-control, authorization, data-exposure, webverse, insecure-direct-object-reference]
platform: Webverse
author: Shivansh Sharma
image:
  path: /assets/images/posts/idor-overdue.webp
  alt: IDOR vulnerability in library checkout system
---

## Overview

This lab demonstrates an **Insecure Direct Object Reference (IDOR)** vulnerability in a public library web application called **Overdue**.

The issue allows an attacker to access other users' checkout records simply by modifying a numeric identifier in the URL, resulting in unauthorized data exposure.

---

## Objective

Identify and exploit an IDOR vulnerability to access restricted checkout records and retrieve the flag.

---

## Reconnaissance

After setting up local domain resolution:

```bash
sudo vi /etc/hosts
```

Add:

```
<ip> overdue.library
```

The application presented a public-facing library system with features such as:

- Browsing books
- Viewing catalog entries
- Checkout-related pages

While testing functionality, direct checkout actions were not available, suggesting restricted operations or missing user privileges.

---

## Discovery of Sensitive Endpoint

While navigating the application, a link led to a checkout record:

```
http://overdue.library/checkouts/8
```

This endpoint exposed detailed information about a specific user's checkout.

---

## Sensitive Data Exposure

The page revealed:

- Borrower identity (name and username)
- Book title and author
- Shelf location
- Checkout and due dates
- Return status
- Private borrower notes

Example:

> "Re-read. The wordplay still holds up."

This is sensitive user-specific data that should not be accessible to other users.

---

## Vulnerability Identification

The endpoint used a predictable numeric identifier:

```
/checkouts/8
```

This is a classic indicator of a potential IDOR vulnerability, where:

- Internal object references (IDs) are exposed
- No authorization checks are enforced
- Access control is missing or improperly implemented

---

## Exploitation

To test for IDOR, the checkout ID was modified:

```
/checkouts/9
```

The application returned a different user's checkout record.

This confirmed that:

- The server does not validate resource ownership
- Any authenticated (or possibly unauthenticated) user can access arbitrary records

By iterating IDs:

```
/checkouts/1
/checkouts/2
/checkouts/3
...
```

it was possible to enumerate multiple users' data.

---

## Proof of Exploitation

Accessing another user's checkout record resulted in exposure of the flag:

```
WEBVERSE{...}
```

This confirms successful unauthorized access to protected resources.

---

## Impact

This vulnerability leads to:

- Unauthorized access to private user data
- Exposure of reading habits and history
- Leakage of internal notes
- Violation of user privacy

In a real-world scenario, this could:

- Breach user confidentiality
- Violate data protection regulations
- Damage organizational trust

---

## Root Cause

The application fails to enforce server-side authorization checks.

Instead of validating ownership, it directly trusts the user-supplied ID:

```
GET /checkouts/<id>
```

There is no verification that:

```python
checkout.user_id == current_user.id
```

---

## Mitigation

### 1. Enforce Authorization Checks

Ensure users can only access their own resources:

```python
if checkout.user_id != current_user.id:
    return 403
```

### 2. Use Indirect Object References

Replace predictable numeric IDs with UUIDs:

```
/checkouts/550e8400-e29b-41d4-a716-446655440000
```

This reduces enumeration risk but does not replace authorization checks.

### 3. Centralized Access Control

Implement middleware to enforce authentication and authorization across all endpoints.

### 4. Avoid Data Overexposure

Return only necessary fields in responses. Sensitive fields like private notes should be restricted.

---

## Real-World Insight

IDOR is one of the most common and dangerous access control vulnerabilities.

It frequently appears in:

- APIs (`/api/user/123`)
- File downloads (`/invoice/456.pdf`)
- Profile pages
- Order histories

High-profile breaches have occurred due to similar issues, where attackers scraped millions of user records through simple ID manipulation.

---

## Final Thoughts

This lab highlights how access control failures are often simple but severe.

No complex payloads or tools were required. Just:

1. Observing URL patterns
2. Modifying a number
3. Repeating the request

That alone was enough to break authorization boundaries.

> In secure application design, **never trust user-controlled identifiers without validation**.