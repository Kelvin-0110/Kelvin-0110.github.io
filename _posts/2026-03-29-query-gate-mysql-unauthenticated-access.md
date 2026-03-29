---
title: "Query Gate: MySQL Unauthenticated Access → Data Exposure"
date: 2026-03-29 22:00:00 +0530
categories: [Hackviser, Basic]
tags: [mysql, database, unauthenticated-access, misconfiguration]
platform: Hackviser
author: Shivansh Sharma
image:
  path: /assets/images/posts/mysql.webp
  alt: MySQL Unauthenticated Access Exploitation
---

## Overview

This lab demonstrates a critical misconfiguration where a MySQL database is exposed without authentication, allowing attackers to access sensitive data directly.

Databases often store highly sensitive information, making such misconfigurations extremely dangerous.

---

## Objective

- Identify exposed database services
- Attempt access without credentials
- Verify unauthorized database access

---

## Reconnaissance

Scan the target for open ports:

```bash
nmap <target_ip>
```

**Output**

```
┌──(root㉿kali)-[/home/kelvin/Desktop]
└─# nmap 172.20.10.79
Starting Nmap 7.98 ( https://nmap.org ) at 2026-03-29 08:18 -0400
Nmap scan report for 172.20.10.79 (172.20.10.79)
Host is up (0.16s latency).
Not shown: 999 closed tcp ports (reset)
PORT     STATE SERVICE
3306/tcp open  mysql

Nmap done: 1 IP address (1 host up) scanned in 3.44 seconds
```

Port **3306 (MySQL)** is open.

---

## Exploitation

Attempt to connect to the MySQL service:

```bash
mysql -u root -h <target_ip> -P 3306
```

**Parameters**

| Flag | Description |
|------|-------------|
| `-u` | Username |
| `-h` | Hostname / Target IP |
| `-P` | Port number |

### Authentication Bypass

The connection was established without requiring any password, indicating a severe misconfiguration.

---

## Proof of Exploitation

- Successfully connected to MySQL as `root`
- No authentication required
- Full access to database environment

Example actions an attacker can perform:

```sql
SHOW DATABASES;
USE <database_name>;
SHOW TABLES;
SELECT * FROM <table_name>;
```

---

## Impact

- Unauthorized access to sensitive data
- Exposure of user credentials and application data
- Potential for data manipulation or deletion
- Risk of full system compromise if credentials are reused

---

## Mitigation

- Enforce strong authentication for MySQL users
- Disable remote root login
- Bind MySQL to localhost if remote access is not required
- Use firewall rules to restrict access to port 3306
- Regularly audit database permissions

---

## Real-World Insight

Exposed databases without authentication are a common issue in misconfigured environments.

Typical risks include:

- Publicly accessible database ports
- Weak or empty passwords
- Overprivileged database users (e.g., root)

Always test:

- Default credentials
- Empty passwords
- Remote access configurations

Databases are high-value targets, and even a simple misconfiguration like this can lead to massive data breaches.