---
title: "Broken Access Control – Unprotected Admin Functionality Leading to Privilege Escalation | Unprotected Admin Functionality"
date: 2026-03-28 04:30:00 +0530
categories: [Web Security]
tags: [
  broken-access-control,
  authorization-bypass,
  admin-panel,
  privilege-escalation,
  portswigger
]
platform: PortSwigger
author: Shivansh Sharma
image:
  path: /assets/images/posts/broken-access-control.webp
  alt: Unprotected Admin Panel Access
---

## Overview

This lab demonstrates a classic case of **broken access control**, where sensitive administrative functionality is exposed without authentication.

An attacker can directly access privileged endpoints and perform critical actions without logging in.

---

## Objective

- Access the admin panel
- Delete the user `carlos`

---

## Reconnaissance

### Step 1: Inspect `robots.txt`


/robots.txt


### Observation

The file reveals a hidden administrative endpoint:


/administrator-panel


### Evidence

![robots.txt output showing admin path](/assets/images/favicons/unprotected-admin-1.png)

This suggests the presence of an admin interface that may not be properly secured.

---

## Exploitation

### Step 2: Access Admin Panel

Navigate to:


/administrator-panel


### Evidence

![Admin panel accessed without authentication](/assets/images/favicons/unprotected-admin-2.png)

No authentication is required to access the panel.

---

### Step 3: Perform Administrative Action

- Accessed the admin interface
- Navigated to user management
- Deleted user `carlos`

---

## Proof of Exploitation

![Lab solved confirmation](/assets/images/favicons/unprotected-admin-3.png)

### Result

- Admin panel accessible without authentication  
- User `carlos` successfully deleted  
- Lab marked as solved  

---

## Impact

This vulnerability can lead to:

- Unauthorized administrative access  
- Data manipulation or deletion  
- Full system compromise  

---

## Mitigation

To prevent this issue:

- Enforce authentication on all admin endpoints  
- Implement proper **Role-Based Access Control (RBAC)**  
- Avoid exposing sensitive paths in `/robots.txt`  
- Use server-side authorization checks (not just hidden URLs)  

---

## Real-World Insight

Relying on hidden endpoints for security is ineffective. Attackers routinely enumerate paths using:

- `/robots.txt`
- JavaScript files
- Directory brute-forcing tools

Security must always be enforced through **proper access control**, not obscurity.

---
