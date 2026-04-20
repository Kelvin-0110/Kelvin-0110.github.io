---
title: "Default Credentials – SSH Misconfiguration Leading to Root Access | Secure Command"
date: 2026-03-29 21:00:00 +0530
categories: [Network Security, Misconfiguration]
tags: [default-credentials, ssh, misconfiguration, authentication-bypass, root-access, linux, hackviser]
platform: Hackviser
author: Shivansh Sharma
image:
  path: /assets/images/posts/ssh.webp
  alt: SSH Default Credentials Exploitation
---

## Overview

This lab demonstrates how weak or default credentials in an SSH service can lead to unauthorized access and full system compromise.

Although SSH is a secure protocol, improper credential management can completely undermine its security.

---

## Objective

- Identify exposed services
- Attempt authentication using default credentials
- Gain unauthorized access to the system

---

## Reconnaissance

Scan the target machine for open ports:

```bash
nmap <target_ip>
```

**Output**

```
┌──(root㉿kali)-[/home/kelvin/Desktop]
└─# nmap 172.20.10.47
Starting Nmap 7.98 ( https://nmap.org ) at 2026-03-29 08:10 -0400
Nmap scan report for 172.20.10.47 (172.20.10.47)
Host is up (0.16s latency).
Not shown: 999 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh

Nmap done: 1 IP address (1 host up) scanned in 3.25 seconds
```

Port **22 (SSH)** is open.

---

## Exploitation

The lab provides the following credentials:

```
hackviser:hackviser
```

However, instead of using only the given credentials, testing common default credentials is always a good approach.

### Authentication Attempts

Tried multiple credential combinations:

```
root:root
root:hackviser
```

### Successful Login

Using:

```
root:root
```

Access was successfully obtained. Example login:

```bash
ssh root@172.20.10.47
```

---

## Proof of Exploitation

- Successfully authenticated via SSH
- Gained root-level access
- No restrictions or additional security controls in place

---

## Impact

- Full system compromise
- Direct root access without privilege escalation
- High risk of persistence and lateral movement
- Complete control over the target machine

---

## Mitigation

- Disable root login via SSH
- Enforce strong, unique passwords
- Use SSH key-based authentication
- Implement fail2ban or rate limiting
- Monitor authentication logs for suspicious activity

---

## Real-World Insight

SSH is widely considered secure due to encryption, but its security depends heavily on configuration.

Common mistakes include:

- Allowing root login
- Using default or weak credentials
- Lack of brute-force protection

Even a secure protocol like SSH becomes a critical vulnerability when basic authentication practices are ignored.

Always test:

- Default credentials
- Username enumeration
- Weak password combinations

These checks often lead to immediate access, as seen in this lab.