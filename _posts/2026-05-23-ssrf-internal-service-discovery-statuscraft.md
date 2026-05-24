---
title: "SSRF – Internal Service Discovery Through Monitor Preview Feature | Statuscraft"
date: 2026-05-23 00:20:00 +0530
categories: [Web Security, SSRF]
tags: [ssrf, internal-service-discovery, localhost-access, port-scanning, network-enumeration, webversepro]
platform: WebVerse
author: Shivansh Sharma
image:
  path: /assets/images/posts/statuscraft.webp
  alt: Statuscraft WebVerse challenge
---

# Lab Link

https://dashboard.webverselabs-pro.com/challenges/inside-line

# Overview

The **Statuscraft** challenge demonstrates a **Server-Side Request Forgery (SSRF)** vulnerability inside a monitor preview feature.

The application allowed users to add monitoring targets and displayed an immediate "first ping" preview before permanently saving the monitor. Instead of validating destinations, the backend fetched arbitrary user-supplied URLs.

Because requests originated from the server itself, internal resources normally inaccessible from external users became reachable.

Using SSRF, internal services were enumerated and hidden resources were discovered.

# Objective

Exploit the monitor preview functionality to access internal services and retrieve the flag.

# Vulnerability Classification Hierarchy

```text
OWASP Category
└── A10: Server-Side Request Forgery (SSRF)
    └── SSRF
        └── Internal Network Enumeration
            └── Localhost Service Discovery
```

# Reconnaissance

Registration was required before accessing monitor functionality.

Registration page:

```text
https://abf6a460-4065-inside-line-f5816.challenges.webverselabs-pro.com/register
```

After registration, monitor creation functionality was available:

```text
https://abf6a460-4065-inside-line-f5816.challenges.webverselabs-pro.com/monitors/new
```

The challenge description mentioned:

```text
The first ping preview it shows after you add a monitor
```

This strongly suggested server-side URL fetching behavior.

Applications that retrieve arbitrary URLs frequently become SSRF candidates.

# Initial Testing

Several localhost targets were tested:

```text
http://127.0.0.1:3000

http://127.0.0.1:5000

http://127.0.0.1:8000

http://127.0.0.1:8080
```

Results:

```text
127.0.0.1:5000 → Reachable

127.0.0.1:8080 → Reachable
```

This confirmed:

- Requests were processed server-side
- Internal services were accessible
- SSRF existed

# Internal Service Discovery

Port enumeration through SSRF revealed multiple internal services.

Observed behavior:

```text
127.0.0.1:5000
```

responded successfully.

Time was spent attempting endpoint discovery and enumeration against the service.

However, no useful content was identified.

Additional testing showed:

```text
127.0.0.1:8080
```

also responding.

Unlike the first service, this endpoint directly exposed sensitive content.

# Exploitation

Requesting:

```text
http://127.0.0.1:8080
```

through the monitor preview functionality returned:

```text
WEBVERSE{.....}
```

The application retrieved internal resources on behalf of the attacker.

# Proof of Exploitation

Attack path:

```text
Monitor Preview Feature
            ↓
Arbitrary URL Submission
            ↓
Server-Side Request
            ↓
Localhost Access
            ↓
Internal Port Enumeration
            ↓
Sensitive Service Discovery
            ↓
Flag Retrieval
```

The server became a proxy into its own internal network.

# Root Cause

The application likely implemented functionality similar to:

```python
response = requests.get(
    user_supplied_url
)
```

without validating destinations.

The application failed to:

- Restrict localhost access
- Block internal network ranges
- Filter sensitive destinations
- Validate request targets

As a result, users could force the server to access internal resources.

# Impact

In real-world environments SSRF can lead to:

- Internal network enumeration
- Cloud metadata access
- API key disclosure
- Credential theft
- Internal application compromise
- Database exposure
- Remote code execution chains

Frequently targeted resources include:

```text
AWS metadata service

GCP metadata service

Internal APIs

Administrative panels

Monitoring systems
```

# Mitigation

## Restrict request destinations

Allow requests only to approved domains:

Bad:

```python
requests.get(user_url)
```

Secure:

```python
if domain in approved_domains:
    requests.get(url)
```

## Block internal IP ranges

Restrict:

```text
127.0.0.0/8
10.0.0.0/8
172.16.0.0/12
192.168.0.0/16
169.254.0.0/16
```

## Resolve hostnames before validation

Perform checks after DNS resolution.

```text
URL
 ↓
DNS Resolution
 ↓
IP Validation
 ↓
Allow/Deny
```

## Isolate outbound requests

Move URL fetching functionality into restricted environments with limited network access.

# Real-World Insight

Features frequently vulnerable to SSRF include:

- URL previews
- Webhooks
- File importers
- PDF generators
- Monitoring systems
- Analytics integrations

Developers commonly assume:

```text
Users only submit public URLs
```

Attackers routinely target:

```text
localhost
internal APIs
cloud metadata
private services
```

Server-side URL fetchers often become internal network proxies when proper restrictions are absent.