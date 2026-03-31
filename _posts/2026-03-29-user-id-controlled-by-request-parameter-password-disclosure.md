---
title: "User ID Controlled by Request Parameter with Password Disclosure"
date: 2026-03-29 13:30:00 +0530
categories: [PortSwigger, Access Control]
tags: [idor, access-control, password-disclosure]
platform: PortSwigger
author: Shivansh Sharma
image:
  path: /assets/images/posts/idor-and-access-contol.webp
  alt: Password Disclosure via IDOR
---

## Overview

This lab demonstrates a **horizontal privilege escalation vulnerability combined with sensitive data exposure**.

The application allows access to other users' profiles via a request parameter and dangerously includes the user's **current password in a prefilled form field**.

---

## Objective

- Retrieve the **administrator's password**
- Log in as administrator
- Delete the user `carlos`

---

## Reconnaissance

Logged in using:

```
wiener:peter
```

After logging in:

- Navigated to the account page
- Observed the URL structure:

```
/my-account?id=wiener
```

This indicates that user accounts are accessed via a **user-controlled parameter**.

---

## Exploitation

### Step 1: Access Other Users

Modified the `id` parameter to access another user:

```
/my-account?id=carlos
```

Confirmed that user accounts are directly accessible.

---

### Step 2: Target Administrator

Changed the parameter to:

```
/my-account?id=administrator
```

Successfully accessed the administrator account page.

---

### Step 3: Extract Password via Interception

Although the password field was masked in the UI, it was still present in the request.

- Intercepted the request using Burp Suite
- Observed the administrator's password in plaintext within the request

![Intercepted Request Showing Admin Password](/assets/images/favicons/admin-password-intercept.png)

Extracted password:

```
v6rzc8cx7puuy6vz3kqk
```

---

## Proof of Exploitation

- Logged in as administrator using the extracted password
- Navigated to the admin panel
- Deleted the user `carlos`

---

## Impact

- Exposure of sensitive credentials (passwords)
- Unauthorized access to privileged accounts
- Full compromise of application access control

> This is a critical vulnerability combining **IDOR** (Insecure Direct Object Reference) and **sensitive data exposure**.

---

## Mitigation

- Never expose passwords in responses (even in masked fields)
- Do not prefill sensitive fields like passwords
- Enforce strict server-side access control checks
- Ensure users can only access their own data
- Follow secure password handling practices (hashing, no retrieval)

---

## Real-World Insight

This is a severe real-world issue.

Even if a password field appears masked in the UI, if it's included in the backend response, it can be easily extracted via interception tools like Burp Suite.

Developers should treat passwords as **write-only data**:

- Store securely (hashed)
- Never send them back to the client under any circumstances