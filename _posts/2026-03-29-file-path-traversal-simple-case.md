---
title: "Path Traversal – Unauthorized File Access via Directory Traversal | Simple Case"
date: 2026-03-29 14:00:00 +0530
categories: [Web Security, File Handling]
tags: [path-traversal, directory-traversal, local-file-inclusion, lfi, unauthorized-file-access, portswigger]
platform: PortSwigger
author: Shivansh Sharma
image:
  path: /assets/images/posts/path-traversal.webp
  alt: File Path Traversal Attack
---

## Overview

This lab demonstrates a basic **file path traversal vulnerability**, where user input is used to fetch files from the server without proper validation.

An attacker can manipulate the file path to access sensitive system files.

---

## Objective

Retrieve the contents of the `/etc/passwd` file.

---

## Reconnaissance

- Accessed the lab and viewed the page source
- Found an image being loaded via a parameterized URL

![Page Source Showing Image Path](/assets/images/favicons/path-source.png)

---

## Exploitation

### Step 1: Identify the Parameter

- Located the image URL:

```
/image?filename=...
```

- Opened the image in a new tab

![Original Image Request](/assets/images/favicons/path-source-1.png)

---

### Step 2: Test Path Traversal

Modified the `filename` parameter to traverse directories:

```bash
https://0a0e0085036e35688631765a005700d9.web-security-academy.net/image?filename=../../../../../etc/passwd
```

---

## Proof of Exploitation

- The server responded with the contents of `/etc/passwd`
- Successfully accessed sensitive system file outside the intended directory

---

## Impact

- Unauthorized access to sensitive files
- Disclosure of system-level information
- Potential stepping stone for further attacks

---

## Mitigation

- Validate and sanitize user input
- Restrict file access to specific directories (whitelisting)
- Use secure APIs to handle file paths
- Normalize paths and block traversal sequences (`../`)

---

## Real-World Insight

Path traversal vulnerabilities are extremely common in poorly secured file-handling systems.

Even simple applications like image loaders can become entry points for attackers if input is not properly validated.

A single vulnerable parameter can expose:

- System files (`/etc/passwd`)
- Configuration files
- Application secrets