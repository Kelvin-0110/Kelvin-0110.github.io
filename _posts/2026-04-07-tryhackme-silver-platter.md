---
title: "Broken Access Control – Credential Leakage to Privilege Escalation | Silver Platter"
date: 2026-04-07 15:30:00 +0530
categories: [Web Security, Access Control]
tags: [broken-access-control, credential-leakage, privilege-escalation, linux, silverpeas, enumeration, tryhackme]
platform: TryHackMe
author: Shivansh Sharma
image: { path: /assets/images/posts/silver-platter.webp, alt: Silver Platter Room }
---

## Overview

This room demonstrates how poor credential management and log exposure can lead to full system compromise. The attack path focuses on web enumeration, credential discovery, and privilege escalation via misconfigured permissions.

---

## Objective

- Enumerate services and web application
- Gain initial access
- Escalate privileges to root

---

## Reconnaissance

### Nmap Scan

```bash
nmap -p- -sV silver-platter.thm
```

**Open Ports:**

- `22` → SSH (OpenSSH 8.9p1)
- `80` → HTTP (nginx 1.18.0)
- `8080` → HTTP Proxy / Web App

---

### Web Enumeration

#### Directory Bruteforce

```bash
dirsearch -u http://silverplatter.thm/
```

**Findings:**

- `/assets` (403)
- `/images` (403)
- `/LICENSE.txt`
- `/README.txt`

#### Virtual Host Discovery

```bash
ffuf -u http://silverplatter.thm -H "Host: FUZZ.silverplatter.thm" -w wordlist.txt
```

**Discovered:**

- `header_popsci`

#### Web Application

- Identified software: **Silverpeas**
- Username enumeration revealed:

```
scr1ptkiddy
```

---

## Exploitation

### Accessing Web Panel

```
http://silverplatter.thm:8080/silverpeas
```

### Credentials Found

```
Username: tim
Password: cm0nt!md0ntf0rg3tth!spa$$w0rdagainlol
```

### SSH Access

```bash
ssh tim@silver-platter.thm
```

---

## Post Exploitation

### Check Sudo Privileges

```bash
sudo -l
```

```
User tim may not run sudo
```

### User Groups

```bash
id
```

```
uid=1001(tim) gid=1001(tim) groups=1001(tim),4(adm)
```

---

### Key Finding: ADM Group Abuse

The `adm` group allows read access to system logs:

- `/var/log/auth.log`
- `/var/log/syslog`

### Extracting Credentials from Logs

Searching logs reveals sensitive data:

```bash
cat /var/log/auth.log
```

**Discovered Password:**

```
_Zd_zx7N823/
```

---

## Lateral Movement

Another user exists:

```
tyler:x:1000:1000:/home/tyler:/bin/bash
```

Switch user:

```bash
su tyler
```

Password:

```
_Zd_zx7N823/
```

---

## Privilege Escalation

### Check Sudo Permissions

```bash
sudo -l
```

If full privileges are available:

```bash
sudo su
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

- Exposure of credentials in logs
- Improper group permissions (`adm`)
- Credential reuse between users
- Full system compromise

---

## Mitigation

- Restrict access to log files
- Avoid storing sensitive data in logs
- Enforce least privilege principle
- Use secure credential storage
- Implement proper monitoring and auditing

---

## Real-World Insight

This scenario reflects real-world environments where:

- Developers log sensitive information for debugging
- System logs become a goldmine for attackers
- Misconfigured group permissions lead to privilege escalation

The attack required no exploit, only proper enumeration and awareness of Linux permissions.

---

## Conclusion

This room highlights how small misconfigurations can chain together into a full compromise. The key takeaway is simple:

> Always check user groups and logs. They often contain more than intended.