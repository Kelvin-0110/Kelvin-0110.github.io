---
title: "Information Disclosure – Sensitive HTML Comment Exposure | Vellichor Press"
date: 2026-05-30 00:00:00 +0530
categories: [A02 - Security Misconfiguration, Information Disclosure]
tags: [webversepro, information-disclosure, source-code-review, html-comments, client-side, security-misconfiguration]
author: Shivansh Sharma
image:
  path: /assets/images/posts/vellichor-press-information-disclosure.webp
  alt: Information Disclosure – Sensitive HTML Comment Exposure | Vellichor Press
---

## Lab Link

Lab: [Vellichor Press](https://dashboard.webverselabs-pro.com/challenges/marginalia)

---

## Overview

Vellichor Press is a literary magazine that recently launched a redesigned website. While the public-facing homepage appears polished and complete, development artifacts were accidentally left behind during deployment.

The challenge revolves around reviewing the HTML source code rather than interacting with visible application functionality.

A developer note embedded within an HTML comment discloses sensitive internal information, including an administrative shortcut and the challenge flag.

This is a classic example of information disclosure caused by exposing development notes in production content.

---

## Objective

Inspect the website source code and identify sensitive information exposed through HTML comments.

---

## Vulnerability Identification

This challenge is primarily an Information Disclosure vulnerability.

### Classification Hierarchy

A02 - Security Misconfiguration
└── Sensitive Information Exposure
└── Source Code Disclosure
└── Sensitive Data in HTML Comments

---

## Reconnaissance

Navigate to the homepage:

```text
https://05de337b-4065-marginalia-4212d.challenges.webverselabs-pro.com/
```

The visible page appears normal.

The challenge description provides two important clues:

```text
draft admin note sits where nobody scrolls
```

and

```text
Some notes are louder in the source than they are on the page
```

These hints suggest examining the underlying HTML source.

---

## Exploitation

### Step 1 - View Page Source

Open the page source using:

```text
Ctrl + U
```

or

```text
Right Click → View Page Source
```

Review the HTML document.

---

### Step 2 - Inspect HTML Comments

Within the source code, a developer comment is discovered:

```html
<!--
  TODO(elias): remove before final launch. Tracking note while the
  reconciliation script runs against the Stripe export — do NOT push
  this to production after Tuesday's deploy.

  admin shortcut (drafts pane):  /staff/admin?token=stripe-recon-2026
  reconcile internal-ref:        WEBVERSE{......}

  — e.l, 2026-03-12, 11:48 ET. Removed by:                       .
-->
```

The comment contains multiple pieces of sensitive information.

---

### Step 3 - Analyze the Disclosure

The exposed comment reveals:

```text
Administrative endpoint
Internal reconciliation details
Development notes
Challenge flag
```

Most importantly:

```text
WEBVERSE{......}
```

The flag is directly disclosed through the page source.

No additional interaction is required.

---

## Proof of Exploitation

### Location

```text
HTML Source Code
```

### Exposed Comment

```html
<!--
Developer note
Administrative shortcut
Internal reference
-->
```

### Flag

```text
WEBVERSE{......}
```

---

## Impact

An attacker can discover:

* Internal endpoints
* Administrative URLs
* API references
* Credentials
* Tokens
* Debug information
* Sensitive operational notes

Although comments are invisible in the rendered page, they remain accessible to anyone who views the source.

In real-world environments, HTML comments have exposed:

* API keys
* Cloud credentials
* Internal IP addresses
* Administrative interfaces
* Temporary passwords
* Source code references

---

## Mitigation

### Remove Development Comments

Development notes should never be deployed to production.

### Review Source Before Release

Perform deployment reviews to identify:

```text
TODO
FIXME
DEBUG
TEMP
INTERNAL
```

artifacts.

### Use Automated Scanning

Static analysis and CI/CD checks can detect sensitive content before deployment.

### Separate Documentation from Production Code

Internal notes should be stored in:

```text
Issue trackers
Project management systems
Documentation repositories
```

rather than application source code.

### Conduct Security Reviews

Review:

```text
HTML
JavaScript
CSS
Source Maps
Comments
```

for exposed information.

---

## Real-World Insight

Information disclosure vulnerabilities frequently originate from development shortcuts that survive into production deployments.

Developers often leave:

```html
<!-- TODO: Remove before launch -->
```

comments containing information that was never intended for public consumption.

Attackers routinely inspect page source, JavaScript files, source maps, and comments during reconnaissance because these locations frequently reveal sensitive information.

The Vellichor Press challenge demonstrates a fundamental lesson:

**If the browser receives the data, the user can read the data. Hidden is not the same as protected.**
