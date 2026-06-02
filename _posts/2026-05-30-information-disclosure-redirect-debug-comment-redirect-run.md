---
title: "Information Disclosure – Redirect Debug Comment Exposure | Redirect Run"
date: 2026-05-30 00:00:00 +0530
categories: [A02 - Security Misconfiguration, Information Disclosure]
tags: [webversepro, information-disclosure, redirects, debug-comments, source-code-review, security-misconfiguration]
author: Shivansh Sharma
image:
path: /assets/images/posts/redirect-run-information-disclosure.webp
alt: Information Disclosure – Redirect Debug Comment Exposure | Redirect Run
----------------------------------------------------------------------------

## Lab Link

Lab: [Redirect Run](https://dashboard.webverselabs-pro.com/challenges/redirect-run)

---

## Overview

Quikpay uses short receipt URLs that redirect customers to a friendly purchase confirmation page.

From a normal user perspective, the redirect happens instantly and nothing appears unusual. However, the intermediate redirect endpoint contains leftover production debugging information that was never removed before deployment.

By interacting directly with the redirect endpoint rather than simply following it in the browser, sensitive internal metadata becomes visible.

This results in information disclosure through exposed debug comments.

---

## Objective

Inspect the receipt shortlink workflow, access the intermediate redirect endpoint, and recover sensitive information exposed through production debug comments.

---

## Vulnerability Identification

### Classification Hierarchy

A02 - Security Misconfiguration
└── Exposed Debug Information
└── Information Disclosure
└── Sensitive Metadata in Redirect Response

---

## Reconnaissance

Navigate to:

```text
https://398ee607-4065-redirect-run-e0f27.challenges.webverselabs-pro.com/receipt
```

The page advertises a demo receipt shortlink:

```html
<div class="url">
  <a href="/r/qp-r4-7821ab">qp.link/r/qp-r4-7821ab</a>
</div>
```

The application encourages users to follow:

```text
/r/qp-r4-7821ab
```

which appears to be a simple redirect mechanism.

---

## Exploitation

### Step 1 - Identify the Redirect Endpoint

Viewing the page source reveals:

```html
<a href="/r/qp-r4-7821ab">qp.link/r/qp-r4-7821ab</a>
```

The receipt flow therefore passes through:

```text
/r/qp-r4-7821ab
```

before reaching the final destination.

---

### Step 2 - Intercept the Request

Capture the request in Burp Suite.

Original request:

```http
GET /receipt HTTP/2
```

Modify it to directly request:

```http
GET /r/qp-r4-7821ab HTTP/2
```

This allows inspection of the intermediate response rather than automatically following the redirect.

---

### Step 3 - Inspect the Response

The response contains a hidden HTML comment:

```html
<!--
  ─── Quikpay redirect debug (production safe? — TODO: strip before launch) ───
  request_token: qp-r4-7821ab
  upstream_status: 200
  upstream_latency_ms: 38
  internal_ref: WEBVERSE{.....}
  reconciliation_window: 24h
  cookie_strip_policy: strict
  note: leave this in until QA signs off on the new tracing pipeline
-->
```

The comment exposes internal operational data that should never be returned to public users.

---

### Step 4 - Extract the Flag

The flag is disclosed directly inside the debug block:

```text
internal_ref: WEBVERSE{.....}
```

Flag:

```text
WEBVERSE{.....}
```

---

## Proof of Exploitation

### Public Endpoint

```text
/receipt
```

### Redirect Endpoint

```text
/r/qp-r4-7821ab
```

### Exposed Debug Comment

```html
<!--
internal_ref: WEBVERSE{.....}
-->
```

### Flag

```text
WEBVERSE{.....}
```

---

## Impact

An attacker can discover:

* Internal identifiers
* Debug metadata
* Operational configuration
* Tracing information
* Backend implementation details
* Sensitive references

In production systems, similar disclosures have exposed:

```text
API keys
Internal hostnames
Session identifiers
Customer references
Database information
Debug credentials
```

Such information frequently assists further attacks.

---

## Mitigation

### Remove Debug Artifacts Before Deployment

Development comments should never be present in production responses.

### Separate Debug and Production Builds

Use dedicated build pipelines that automatically strip:

```text
TODO
DEBUG
TRACE
DEV
```

content.

### Avoid Exposing Internal Metadata

Information such as:

```text
internal_ref
latency
backend status
tracing identifiers
```

should remain server-side.

### Review Intermediate Responses

Security testing should include:

```text
Redirect targets
API responses
Error pages
Source code
Comments
```

rather than only visible page content.

### Conduct Release Reviews

Perform final production audits for:

```text
Comments
Debug routes
Temporary features
Diagnostic output
```

before deployment.

---

## Real-World Insight

Attackers often inspect pages that users rarely see directly:

* Redirect responses
* Error handlers
* Intermediate APIs
* Debug endpoints

These locations frequently contain leftover development information because teams focus on the visible application experience rather than every response returned by the server.

The Redirect Run challenge demonstrates a common security lesson:

**Sensitive information does not need to be visible to be exposed. If it is sent to the browser, it is available to the user.**
